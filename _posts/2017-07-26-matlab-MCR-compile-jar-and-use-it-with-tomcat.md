---  
layout: post  
title:  "Matlab Compiler Runtime与tomcat应用于matlab和java混编及网站开发"  
categories: MCR  
tags:  Matlab MCR Tomcat JavaWeb  
author: bluer  
mathjax: true  
---  
  
* content  
{:toc}  
  
Matlab Compiler Runtime（MCR）  
“MATLAB Runtime 是一组独立的共享库，可用于**在未安装 MATLAB 的计算机上执行编译后的 MATLAB 应用程序或组件**。 MATLAB, MATLAB Compiler与 MATLAB Runtime 配合使用，可以快速、安全地创建和分发数值应用程序或软件组件。”    
MCR还**可以配合matlab的Library Compile package编译matlab代码为c++、java、python等语言**。    
所以我们可以把使用library compile package**把matlab程序编译成jar，然后配合tomcat发布javaweb网站。**    
之前使用matlab程序做网站的做法都是在server上装一个matlab，web的java后台调用系统命令启动matlab，执行matlab脚本，存储结果web在使用java代码调用结果文件。这样做网站响应速度会很慢，而且占web server的很大资源。    
使用mcr要好很多。  
  
下边我们来动手实践吧（matlab2017a，windows8.1，server：ubuntu14.04）加粗显示的都是趟过的报错坑：  





首先我们要明确我们的任务，我们的最终目标是要把依赖第三方工具包（一个深度信念网络包）的matlab程序编译成jar（涉及一个很复杂的第三方包，不知道能不能成），混合到web的java后台代码中。（成的话，我们就算实践了mcr最复杂的用法）      
所以：      
1. 在windows本机上使用library compile package编译matlab代码  
2. 先在windows平台上配置环境试验是否可以运行jar  
3. 在linux服务器上配置环境试验是否可以运行jar  
4. 编写网站配合tomcat上传运行jar  
  
## 1、在windows本机上使用library compile package编译matlab代码    
首先只要使用matlab编译java，都是要配置和matlab相对应的jdk的    
在Matlabd的Command Windows中输入version -java    
没错，最新的matlab2017a也是 'Java 1.7.0_60-b19 with Oracle Corporation Java HotSpot(TM) 64-Bit Server VM mixed mode'    
所以装1.8的，可以下载一个1.7zip版，配置环境变量    
打开cmd 分别输入命令java -version和javac -version分别验证编译和运行版本。    
**javac没配置好，编译jar包是要报错的。**    
**jdk版本版本不对是要报错的。**    
**编译jar时内存不足也是会报错的**：Failed to embed unzip in your applicationUpdate resource failed: 00000000CE555B5，Failed to embed installer splash screen ...toolbox\compiler\Resources\default_splash.png. Update resource failed: 0000000037E55CB0    
编写主函数脚本，这个脚本要做到对外提供一个接口之类的作用，比如我们这个文件是根据用户输入的文件路径，生成深度信念网络跑出来的特征：  
```matlab  
	function []=getDbnFeat(file_path)  
    [tits, seqs]=fastaread(file_path);  
    num=length(tits);  
	len=length(char(seqs(1)));  
  
    Y=zeros(num,1);  
    X=zeros(num,len*4);  
    %onehot  
    for i=1:num  
  
        str=char(tits(i));  
        temp=strsplit(str,'|');  
        label=char(temp(2));  
        if label=='1'  
            Y(i)=0;  
        else  
            Y(i)=1;  
        end  
  
  
        img=zeros(len,4);  
        for j=1:len  
            seq=char(seqs(1,i));  
            switch seq(j)  
                case 'A'  
                    img(j,:)=[0,0,0,1];  
                case 'C'  
                    img(j,:)=[0,0,1,0];  
                case 'G'  
                    img(j,:)=[0,1,0,0];  
                case 'U'  
                    img(j,:)=[1,0,0,0];  
                case 'T'  
                    img(j,:)=[1,0,0,0];  
            end  
        end  
        X(i,:)=img(:);  
  
    end  
  
    inputs = X;  
    targets = Y;  
    load('dbn.mat');  
    feat=dbn.getFeature(inputs);  
    feat_label=[feat,targets];  
    csvwrite('feat_mid.csv',feat_label);  
    S = fileread('feat_mid.csv');  
    feat_dim=100;  
    attribute='';  
    for i=1:feat_dim  
        attribute=[attribute,'@attribute Fea',int2str(i),' numeric',char(10)];  
    end  
    header=['@relation m6a_dbn',char(10),attribute,'@attribute class {0,1}',char(10),'@data',char(10)];  
  
    S = [header,  S];  
    FID = fopen('feat.arff', 'w');  
    if FID == -1, error('Cannot open file'); end  
    fwrite(FID, S, 'char');  
    fclose(FID);  
end  
```  
LibraryCompiler在matlab的APP一栏下     
 ![](http://bbs.malab.cn/data/attachment/forum/201707/19/095650vkh886ak5ah2y462.jpg)    
选择要编译的语言，点击+好选择编写好的主函数脚本，更改生成的jar名    
![](http://bbs.malab.cn/data/attachment/forum/201707/19/095651b88y7bqg1bgezhk7.jpg)    
library compile package会自动选择函数脚本依赖的文件，但是真得会有可能并没有那么智能，特别是我们这种复杂的第三方包。    
![](http://bbs.malab.cn/data/attachment/forum/201707/19/095651p14wioijsjjlfgwl.jpg)    
点击右上角package对号按钮，开始一段漫长的编译。    
tips：  
1. 不要涉及mex等平台特定性文件，默认编译的jar文件是平台不相关的，这样部署到linux网站才有可行性，所以不要涉及平台特定性文件。  
2. 函数内涉及调用mat怎么办，没事，library compile package会智能关联依赖，一并打包。  
3. 函数外涉及mat调用咋办，就是java调用mat，这个有一个叫[JMatIO](https://sourceforge.net/projects/jmatio/)的可以用。  
4. 涉及二次存储的（比如本例中先使用matlab自带的csvwrite存储成csv文件，再读取csv略改变成arff文件），要注意，library compile package会自动把中间文件打包进jar中，这样会导致程序出错，因为每次调用jar文件，程序都是对jar内部存储中间文件做修改，方法是打包时删点文件夹中的中间文件和结果文件，然后在打开library compile package，这样library compile package就不会自动关联依赖，**不删除直接在依赖列表中删除会报错**。  
点击这个启动library compile package  
5. **编译成功后，运行报错可能会提示少什么类，在第三方包中找到包含相应类的m文件，添加到依赖文件列标中。**  
编译成功后会生成三个文件夹：  
“-for_redistribution包含用于安装应用程序和MATLAB Runtime的文件，可以将这份文件复制到没有安装Matlab的电脑上，安装该文件，在安装过程中会提示要安装独立的共享库MATLAB Runtime，安装之后不需要Matlab也可以运行编译后的Matlab的程序或者元件(The MATLAB Runtime is a standalone set of shared libraries that enables the execution of compiled MATLAB applications or components on computers that do not have MATLAB installed)。  
-for_redistribution_files_only文件夹包含应用程序的重新发布所需的文件。这些文件可以分发到那些有MATLAB或者有 MATLAB Runtime 的用户的电脑上。  
-for_testing文件夹包含创建的所有由MCC创建的文件，像二进制文件和jar，头和源文件，使用这些文件来测试安装。 -PackagingLog.txt是由编译器生成的日志文件。 如果.jar库中的内容不熟悉也可以找到doc下的html文件夹，打开index.html,里面有Javadoc参考资料。” --marine0131  
我们使用for_redistribution_files_only或者for_testing的jar文件进行下一步。  
  
## 2、先在windows平台上配置环境试验是否可以运行jar  
**请配置mcr前一定要把这个官方文档读透，虽然并不能解决很多出错，但是会让你排除很多不对的博客的胡说八道**  
https://cn.mathworks.com/help/compiler/install-the-matlab-runtime.html  
https://cn.mathworks.com/help/compiler_sdk/java/configure-your-java-environment.html  
其中有一点值得注意的是，没装matlab的可以装一个800m到1G的mcr installer，装过matlab（10G）的是不用装mcr的，除非你想使用不同版本的matlab运行时。很多帖子因为出错都同时装了matlab和mcr，虽然最后也成功了，但是都是画蛇添足。  
matlab命令行中执行 mcrinstaller 会出现存放mcrinstaller的目录'C:\Program Files\MATLAB\R2017a\toolbox\compiler\deploy\win64\MCRInstaller.exe'我感觉这是让你拷给别人以便对方没有matlab环境能使用你的程序。如果你没有装matlab，应该 去官网直接下载mcr installer。  
确保环境变量中的Path中包含C:\Program Files\MATLAB\R2017a\runtime\win64;C:\Program Files\MATLAB\R2017a\bin，特别是前者\runtime\win64存放了mcr运行需要的dll库。  
确保不同机器上的matlab版本或者mcr版本一致，否则行不通。  
  
在eclipse中新建java项目导入生成的jar包和MATLAB的安装目录或者MATLAB Runtime的安装目录下的toolbox\javabuilder\jar\win64\javabuilder.jar  
编写测试代码：  
  
```java  
package test;  
  
import java.io.FileNotFoundException;  
import java.io.IOException;  
import com.mathworks.toolbox.javabuilder.MWException;  
import getDbnFeat.*;  
  
public class test {  
  
	public static void main(String[] args) throws FileNotFoundException, IOException {  
		  
		DbnFeat matlab;  
		String path=args[0];  
		try {  
			matlab = new DbnFeat();  
			matlab.getDbnFeat(path);  
		} catch (MWException e) {  
			// TODO Auto-generated catch block  
			e.printStackTrace();  
		}  
		  
	}  
  
}  
```	  
能注意避开以上可能出现的错误，基本就能正常执行。  
  
## 3、在linux服务器上配置环境试验是否可以运行jar  
由于平台差异性，在server上配置还是有很多差异的。  
我再server上配置了jdk1.7，并把tomcat配置从1.8改为了1.7.  
重装了matlab2017a  
装在server端装matlab有几点：  
1、**server自身存储一般都很小，linux的matlab默认是装在/usr/local下的，装到一般基本会报内存不足，必须装在/mnt外接硬盘上**  
要使用配置文件装：  
创建文件夹 ~/matlabSetup   
在其下创建两个文件 installer_input2.txt and activate.ini.  
  
In installer_input2.txt:  
  
	destinationFolder=Path/To/Destination/Folder/  
	fileInstallationKey=xxxxx-xxxxx-xxxxx-xxxxx-xxxxx-xxxxx-xxxxx-xxxxx-xxxxx  
	agreeToLicense=yes  
	outputFile=/tmp/matlab.log  
	mode=silent  
	activationPropertiesFile=~/matlabSetup/activate.ini  
	licensePath=~/matlabSetup/license.lic  
	product.MATLAB  
	  
product.MATLAB这行是设置只安装matlab主程序，默认全安装可以去掉这行。  
  
In activate.ini:  
  
	isSilent=true  
	activateCommand=activateOffline  
	licenseFile=~/matlabSetup/license.lic  
  
license.lic文件为crack文件夹中的license-standlone.lic  
  
ssh中执行：  
  
	cd /media/mathworks  
	sudo ./install  -mode silent -inputFile ~/matlabSetup/installer_input2.txt   
  
即可。  
  
配置jdk，确保matlab命令行中输入getenv JAVA_HOME能得到正确的javahome路径。  
 和windows一样装有matlab就不用装mcr，但是必须让程序找到runtime库，不配置直接运行jar会报这个最普遍出现的错误：  
Failed to find the library libmwmclmcrrt.so.XXXX  
很多人又装了matlab又装了mcr，然后配置了LD_LIBRARY_PATH解决了问题，官方文档也说要配置LD_LIBRARY_PATH，但是啥也没说一笔带过。  
其实缺什么补什么就可以，[b]使用find / -name libmwmclmcrrt 找到这个so链接文件，然后把路径添加到LD_LIBRARY_PATH[/b]，和windows的Path中的runtime\win64是一个作用。  
LD_LIBRARY_PATH不能随便改，这个东西是先与系统默认的库文件加载自定义库文件的，如果先加载了不兼容C库的文件之类的，整个系统都会直接崩掉，并启动不了！（相当于删库跑路，之前有过惨痛教训）  
改了LD_LIBRARY_PATH，ssh上基本就可以运行jar了。  
  
## 4、编写网站配合tomcat上传运行jar  
接下来就是利用jar配合java编写web后台，然后部署网站了，  
但是，linux的ssh里能运行不一定tomcat里能运行，**java后台代码里要利用ProcessBuilder的environment 方法再设置一遍LD_LIBRARY_PATH**  
否则web还会报Failed to find the library libmwmclmcrrt.so.XXXX的错。  
```java  
ProcessBuilder pb = new ProcessBuilder(new String[] { "java", "-classpath", ".:./javabuilder.jar:./getDbnFeat.jar","test","in.txt" }).directory(new File(directory));  
Map<String, String> env = pb.environment();  
  
env.put("LD_LIBRARY_PATH", "/mnt/Matlab/runtime/glnxa64");  
Process process=pb.start();  
process.waitFor();  
```  
  
最后不知道为啥windows下用eclipse打的调用matlab jar文件和javabuilder.jar生成特征的java代码的jar包，linux上回报错，classpath路径也改成了相对路径，我使用了直接使用linux本地javac编译java执行的方式：  
  
	javac -classpath ./javabuilder.jar:./getDbnFeat.jar  ./test.java  
	java -classpath .:./javabuilder.jar:./getDbnFeat.jar test p1_n1_H.txt  
第一行使用javac编译， classpath参数设置依赖（windows用;分割多个依赖包，linux使用:分割多个依赖包）  
第二行利用生成的class文件对输入序列编码特征。