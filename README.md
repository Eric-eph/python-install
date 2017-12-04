# python-install
install something apk
#-*- coding: utf-8 -*-
import os
import re,time
from weq import download

time1= time.time()#获取当前时间戳
def stop_port():
    os.popen("adb devices")
    str=os.popen("netstat -aon|findstr 5037").read()
    tup=re.split("\\s+",str)
    if tup[5]:
        pid1=tup[5]
        if pid1!=0:
            os.popen("taskkill /f /pid:"+pid1)
    else:
        print "结束端口",str
    os.popen("adb devices")
               # 结束端口占用进程
def get_file_list(filepath1,originalpathlistlevel,originalfilelistlevel):
    #获取路径和文件列表，返回形式为[[arg1],[arg2]]
    cmd="adb shell ls -al "+ "'"+filepath1+ "'"
    output=os.popen(cmd).read()
    if  "No such file or directory" not in output:
        List=re.split('\n',output)
        if List:
            for outstr in List[:-1]:
                string=map(lambda x: x.strip(), list(re.findall(".*(\d{4}-\d{2}-\d{2} \d{2}:\d{2}) (.*)", outstr)[0]))
                typecode=outstr[0]
                if typecode=="d":
                    path=filepath1+string[1]+"/"
                    originalpathlistlevel.append(path)#获取当前一级目录
                elif typecode=="-":
                    filename=filepath1+string[1]
                    originalfilelistlevel.append(filename)#获取当前一级文件
                elif typecode=="*":
                    continue
                else:
                    print "特例1",cmd,output
        else:
            originalpathlistlevel.append(filepath1)
        return originalpathlistlevel,originalfilelistlevel
    else:
        print "特例2",filepath1,output

def get_newfile_list(file_path1,originalpathlist,originalfilelist):
    #获取编辑后的路径和文件列表，返回形式[[arg1],[arg2]]
    cmd="adb shell ls -al "+ "'"+ file_path1 + "'"
    output=os.popen(cmd).read()
    List=re.split('\n',output)
    if List:
        pathlist=[]
        filelist=[]
        for outstr in List[:-1]:
            typecode=outstr[0]
            #获取文件格式标识符，为d或者-
            if typecode=="d" or typecode=="-":
                string=map(lambda x: x.strip(), list(re.findall(".*(\d{4}-\d{2}-\d{2} \d{2}:\d{2}) (.*)", outstr)[0]))
                modifytime=string[0]
                mtime=filter(lambda x:x.isdigit(),modifytime)
                #对文件时间格式进行规范
                localtime2=time.strptime(mtime,'%Y%m%d%H%M')
                time2= time.mktime(localtime2)
                if (time1-time2)<60:#编辑时间在安装时间之后
                    if typecode=="d":
                        path=file_path1+string[1]+"/"
                        pathlist.append(path)
                        # if path not in originalpathlist:
                        #     # print u"创建的文件夹",path,modifytime
                        # else:
                            # print u"变动的文件夹",path,modifytime
                    elif typecode=="-":
                        filename=file_path1+string[1]
                        filelist.append(filename)
                        if filename not in originalfilelist:
                            print u"创建的文件",filename,modifytime
                        else:
                            print u"变动文件",filename,modifytime
            else:
                print "特例3",file_path1,outstr
        return pathlist,filelist
    else:
        print u"空文件夹",filepath1
def return_allapk():
        '''
        返回手机上已经安装的所有包名
        :param appinfo:
        :return:
        '''
        packages_cmd = "adb shell pm list packages -3"
        strout = os.popen(packages_cmd).read()
        packages = strout.split('\r\r\n')
        return packages[0]

test = download()
apklist=test.get_packagename()
for apk in apklist:
    filepath1="sdcard/"
    originalpathlistlevel1=[]
    originalfilelistlevel1=[]
    originalpathlistlevel2=[]
    originalfilelistlevel2=[]
    stop_port()
    print u"开始扫描本地文件，请稍等"
    level1path=get_file_list(filepath1,originalpathlistlevel1,originalfilelistlevel1)[0]
    for path in level1path:
        get_file_list(path,originalpathlistlevel2,originalfilelistlevel2)
    filepath2=filepath1+"android/data/"
    originalpathlistlevel3=[]
    originalfilelistlevel3=[]
    originalpathlistlevel4=[]
    originalfilelistlevel4=[]
    level3path=get_file_list(filepath2,originalpathlistlevel3,originalfilelistlevel3)[0]
    for path in level3path:
        get_file_list(path,originalpathlistlevel4,originalfilelistlevel4)
    cmd='adb install -r ' + 'F:\\apk\\zwf1\\' + apk + '.apk'
    print cmd
    print u"安装",apk,u"中，请稍等。。。"
    output=os.popen(cmd).read()
    if apk in return_allapk() and (re.findall('Success', output) or 'rm: /data/local/tmp' in output[0]):
        raw_input (u"安装完成，请手动运行各项功能，完成后输入回车开始对比")
        stop_port()
        level1 = get_newfile_list(filepath1,originalpathlistlevel1,originalfilelistlevel1)
        for pathlevel1 in level1[0]:
             level2=get_newfile_list(pathlevel1,originalpathlistlevel2,originalfilelistlevel2)
             contentstr = ''.join(level2[0]+level2[1])
             if pathlevel1 not in contentstr:
                print "创建路径为",pathlevel1
             for pathlevel2 in level2[0]:
                 s = get_newfile_list(pathlevel2,originalpathlistlevel2,originalfilelistlevel2)
                 contentstr = ''.join(s[0]+s[1])
                 if pathlevel2 not in contentstr:
                     print "创建路径为",pathlevel2
        level3 = get_newfile_list(filepath2,originalpathlistlevel3,originalfilelistlevel3)
        for pathlevel3 in level3[0]:
             level4=get_newfile_list(pathlevel3,originalpathlistlevel4,originalfilelistlevel4)
             for pathlevel4 in level4[0]:
                 get_newfile_list(pathlevel4,originalpathlistlevel4,originalfilelistlevel4)

        raw_input(u"对比结束,输入回车卸载软件并返回卸载残留")
        cmd = "adb uninstall " +  apk
        output = os.popen(cmd).read()
        if apk not in return_allapk():
            print apk,"卸载成功"
            uninstallpath1=[]
            uninstallfile1=[]
            uninstallpath2=[]
            uninstallfile2=[]
            uninstallpath3=[]
            uninstallfile3=[]
            uninstallpath4=[]
            uninstallfile4=[]
            if level1[0]:
                for path in level1[0]:
                    cmd="adb shell ls -al "+path
                    output=os.popen(cmd).read()
                    if "No such file or directory" not in output:
                        unlevel1=get_file_list(path,uninstallpath1,uninstallfile1)
                        if  path not in originalpathlistlevel1:
                            print u"残留路径",path
                        if unlevel1[0]:
                            for path2 in unlevel1[0]:
                                if (path2 not in originalpathlistlevel2) and (path2 in level2[0]):
                                    print u"残留路径",path2
                        if unlevel1[1]:
                            for file in unlevel1[1]:
                                if (file not in originalfilelistlevel2) and (file in level2[1]):
                                    print u"残留文件",file
            uninstallfile=get_file_list(filepath1,uninstallpath2,uninstallfile2)[1]
            if uninstallfile:
                for file in uninstallfile:
                    if (file in level1[1]) and (file not in originalfilelistlevel1) :
                        print u"残留文件",file
            if level3[0]:
                for path in level3[0]:
                    cmd="adb shell ls -al "+path
                    output=os.popen(cmd).read()
                    if "No such file or directory" not in output:
                        unlevel3=get_file_list(path,uninstallpath3,uninstallfile3)
                        if path not in originalpathlistlevel3:
                            print u"残留路径",path
                        if unlevel3[0]:
                            for path3 in unlevel3[0]:
                                if (path3 not in originalpathlistlevel4) and (path3 in level4[0]):
                                    print u"残留路径",path3
                        if unlevel3[1]:
                            for file in unlevel3[1]:
                                if (file not in originalfilelistlevel4) and (file in level4[1]):
                                    print u"残留文件",file
            uninstallfile2=get_file_list(filepath1,uninstallpath4,uninstallfile4)[1]
            if uninstallfile2:
                for file in uninstallfile2:
                    if (file in level3[1]) and (file not in originalfilelistlevel3):
                        print u"残留文件",file
            raw_input(u"对比完成,输入回车进入下一个应用")
    else:
        print apk,"安装失败"
print "程序结束"
