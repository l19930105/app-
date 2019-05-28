#/user/bin/python
#encoding:utf-8

import os
import time
import csv

import subprocess
import re
import easygui as g
import sys

si = subprocess.STARTUPINFO()
si.dwFlags |= subprocess.STARTF_USESHOWWINDOW
'''
本脚本实现的为：在某一测试场景下，监控APP的专项数据："CPU", "内存", "电量", "流量"
专项数据分析：
CPU：
  在一个时间段如20分钟cpu在40%稳定，是在可接受范围内的。
  若一直在80%以上，就是有问题的。
内存：
  连续测试两个小时，看内存变化的曲线范围，如果有几百M的范围波动变化，就需要关注跟踪问题
电量：
    设定一个场景，在半个小时查看手机电量消耗情况，短时间无法取到可供参考分析的值
流量：
  eth0和eth1是两个网卡，两个网卡和就是流量值
  将最后一次的值减去第一次值获得操作后的消耗流量
    1. 与竞品进行分析  -- 消耗流量多需分析
    2. 历史版本比较分析
      v1.2 20M
      v1.3 30M
'''

'''
adb shell top可选的参数如下:
    -m num  Maximum number of processes to display. 最多显示多少个进程
    -n num  Updates to show before exiting.  刷新次数 
    -d num  Seconds to wait between updates. 刷新间隔时间（默认5秒）
    -s col  Column to sort by (cpu,vss,rss,thr). 按哪列排序 
    -t      Show threads instead of processes. 显示线程信息而不是进程
    -h      Display this help screen.  显示帮助文档 

    adb命令如下：
    adb shell top -n 1|findstr com.xiaomi.shop
'''
# 控制类
class Controller(object):
    def __init__(self, count):
        self.counter = count
        self.alldata = [("timestamp", "CPU", "内存", "电量", "流量", "pid", "uid", "CPU异常信息", "内存异常信息", "activity")]
        self.cpuvalue = ""   # cpu
        self.memvalue = ""   # 内存
        self.power = ""      # 电量
        self.receive = 1     # 收到
        self.transmit = 1    # 传输
        self.receive2 = 1
        self.transmit2 = 1
        self.activity = ""   # activity
        self.pid = ""        # pid
        self.uid = ""        # uid
        self.max_cpu = 70     # cpu预警值
        self.max_mem = 300   # mem预警值
        self.abnormal_cpu_msg = ""  # cpu异常信息
        self.abnormal_mem_msg = ""  # mem异常信息
        self.msg = g.enterbox(msg="请输入你要监控的使用场景", title="监控APP专项数据")
        # 输入包名
        self.appPackage = "com.xiaomi.shop"
        # self.appPackage = "cn.art.app"

    # 获取当前设备名
    def get_devices(self):
        devices = []
        result = subprocess.Popen("adb devices", shell=True, stdout=subprocess.PIPE,
                                  stderr=subprocess.PIPE).stdout.readlines()
        # print(result[1].decode())
        # print(">>>命令结果 len(result)" + str(len(result)))
        if len(result) - 2 == 1:
            for line in result[1:]:
                devices.append(line.strip().decode())
            print("当前设备名为： " + str(devices[0].split()[0]))
            return devices[0].split()[0]
        else:
            # 如果没有设备弹窗提示，并退出程序
            g.msgbox(image='Warning.gif', msg="当前没有移动设备或不止一个设备!!!")
            sys.exit(0)

        return devices[0]

    # 获取包名
    def getpackagename(self):
        pattern = re.compile(r"[a-zA-Z0-9\.]+/.[a-zA-Z0-9\.]+")
        package = subprocess.Popen("adb shell dumpsys activity | findstr  mFocusedActivity", shell=True,
                                   stdout=subprocess.PIPE, stderr=subprocess.PIPE).stdout.read()
        package = (str(package))
        packagename = pattern.findall(package)[0].split('/')[0]
        print("测试app包名为： " + str(packagename))
        return packagename

    # 获取activity
    def getactivity(self):
        pattern = re.compile(r"[a-zA-Z0-9\.]+/.[a-zA-Z0-9\.]+")
        package = subprocess.Popen("adb shell dumpsys activity | findstr  mFocusedActivity", shell=True,
                                   stdout=subprocess.PIPE, stderr=subprocess.PIPE).stdout.read()
        package = (str(package))
        activity = pattern.findall(package)[0].split('/')[1]
        print("当前操作activity包名为： " + str(activity))
        return activity

    # 获取pid
    # PID是进程的身份标识，程序一旦运行，就会给应用分配一个独一无二的PID
    def get_pid(self):
        pid_list = []
        cmd = 'adb -s ' + self.get_devices() + ' shell ps |findstr ' + self.getpackagename()
        pid_info = subprocess.Popen(cmd, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE).stdout.readlines()
        if len(pid_info) >= 1:
            pid_list.append(int(pid_info[0].split()[1]))
        return str(pid_list[0])

    # 获取uid
    # UID 指用户ID 为了实现数据共享，android为每个应用几乎都分配了不同的UID，不像传统的linux，每个用户相同就为之分配相同的UID
    def get_uid(self):
        uid_list = []
        cmd = 'adb -s ' + self.get_devices() + ' shell cat  /proc/' + self.get_pid() + '/status'
        uid_info = subprocess.Popen(cmd, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE).stdout.readlines()
        if len(uid_info) >= 1:
            uid_list.append(int(uid_info[6].split()[1]))
        return str(uid_list[0])


    # 百分数转为int
    def percent_to_int(self,string):
        if "%" in string:
            newint = int(string.strip("%")) / 100
            return newint
        else:
            print("你输入的不是百分比！")

    # 单次测试过程
    def testprocess(self):
        # 获取pid和uid
        self.pid = self.get_pid()
        self.uid = self.get_uid()

        # 获取CPU
        result = os.popen('adb shell "dumpsys cpuinfo | grep com.xiaomi.shop"')
        # result = os.popen('adb shell "dumpsys cpuinfo | grep cn.art.app"')
        for line in result.readlines():
            self.cpuvalue = line.split("%")[0]
            break  # 若找出多个内存占用值，取第一个取到就停止，否则会取到最后的0

        # 如果cpu超过预警值 提示
        if float(self.cpuvalue) > self.max_cpu:
            # print("cpu爆表啦，快去看看" + str(self.cpuvalue))
            # 记录异常信息
            self.abnormal_cpu_msg = "当前cpu值超过预警值" + str(self.max_cpu) + "%"
            # CPU超过预警就记录当前activity
            self.activity = self.getactivity()

        # 获取内存和cpu(cpu已去除）
        # result = os.popen("adb shell top -n 1|findstr com.xiaomi.shop")
        result = os.popen("adb shell top -n 1|findstr %s" %self.appPackage)
        for line in result.readlines():
            if "fg" in line:
                # 将所有空行换成#
                line = "#".join(line.split())
                # self.cpuvalue = line.split("#")[2]
                # self.cpuvalue = self.percent_to_int(self.cpuvalue)
                # print("统一命令获取的CPU  " + str(self.cpuvalue))
                # # 如果cpu超过预警值 提示
                # if self.cpuvalue > self.max_cpu:
                #     print("cpu爆表啦，快去看看" + str(self.cpuvalue))
                #     self.activity = self.getactivity()
                #     print("activity>>>>>" + str(self.activity))
                self.mem = line.split("#")[6]
                self.memvalue = str(round(int(self.mem[:-1]) / 1024))
                # print(">>>self.memvalue: " + self.memvalue)
                if int(round(int(self.mem[:-1]) / 1024)) > self.max_mem:
                    # CPU超过预警就记录当前activity
                    self.activity = self.getactivity()
                    # 记录mem异常信息
                    self.abnormal_mem_msg = "当前mem值超过预警值" + str(self.max_mem) + "M"

        # 手机连接到电脑，默认为充电状态
        # 切换手机电池为非充电状态  --2 为充电状态
        os.popen("adb shell dumpsys battery set status 1")
        # 执行获取电量的命令
        powerData = os.popen("adb shell dumpsys battery")
        # 获取电量的level
        for line in powerData:
            if "level" in line:
                # 格式化电量值--去除换行
                self.power = line.split(":")[1].replace("\n", "")

        # 小米商城 --
        # result = os.popen('adb shell "ps | grep com.xiaomi.shop"')
        result = os.popen('adb shell "ps | grep %s"'%self.appPackage)
        # 获取进程ID
        pid = result.readlines()[0].split(" ")[4]
        # print(">>>>>pid: " + str(pid))
        # 获取进程ID使用的流量
        traffic = os.popen("adb shell cat /proc/" + pid + "/net/dev")
        for line in traffic:
            if "eth0" in line:
                # 将所有空行换成#
                line = "#".join(line.split())
                # 按#号拆分，获取收到和发出的流量
                self.receive = line.split("#")[1]
                print("receive为：" + str(self.receive))
                self.transmit = line.split("#")[9]
                print("transmit为：" + str(self.transmit))
            elif "eth1" in line:
                # 将所有空行换成#
                line = "#".join(line.split())
                # 按#号拆分，获取收到和发出的流量
                self.receive2 = line.split("#")[1]
                # print("222receive2为：" + str(self.receive2))
                self.transmit2 = line.split("#")[9]
                # print("222transmit为：" + str(self.transmit2))

        # 计算所有流量之和
        alltraffic = int(self.receive) + int(self.transmit) + int(self.receive2) + int(self.transmit2)
        # print("所有流量之和为： " + str(alltraffic))
        # 按KB计算流量值
        alltraffic = alltraffic / 1024
        alltraffic = alltraffic / 1024
        # 保留小数点后两位
        alltraffic = round(alltraffic, 2)
        alltraffic = str(alltraffic)


        # 获取当前时间
        currenttime = self.getCurrentTime()
        # 将获取到的数据存到数组中
        self.alldata.append((currenttime, self.cpuvalue, self.memvalue, self.power, alltraffic, self.pid, self.uid, self.abnormal_cpu_msg, self.abnormal_mem_msg, self.activity))

        # print(">>>>>cpuvalue " + self.cpuvalue)
        # print(">>>>>mem " + self.mem)
        # print(">>>>>memvalue " + self.memvalue)
        # print(">>>>>power " + self.power)
        # print(">>>>>消耗流量为： " + str(alltraffic))

    # 多次执行测试过程
    def run(self):
        # 当执行次数大于0则执行单次测试过程，并次数-1
        while self.counter > 0:
            self.testprocess()
            self.counter = self.counter - 1
            time.sleep(3)

    # 获取当前的时间戳
    def getCurrentTime(self):
        currentTime = time.strftime("%Y-%m-%d %H:%M:%S", time.localtime())
        return currentTime

    # 数据的存储
    def SaveDataToCSV(self):
        # 获取当前时间 可确保测试报告文件不重名
        now = time.strftime("%Y-%m-%d %H_%M_%S")
        csvName = "【" + self.msg + "】" + now + ' APPstatus.csv'
        # 定义一个csv文件 --记录数据
        csvfile = open(csvName, 'w', newline='')
        writer = csv.writer(csvfile)
        # 写数据
        writer.writerows(self.alldata)
        csvfile.close()

if __name__ == '__main__':
    # 存入程序开始时间
    start = time.clock()

    # 实例化controller类 并执行20次  20次-140秒
    controller = Controller(20)
    # 获取当前系统是否有且只有一个移动设备
    controller.get_devices()
    # 如果上述不是一个移动设备，系统退出

    # 获取监控app的包名
    controller.getpackagename()
    # 获取监控app的activity
    controller.getactivity()

    # 跑起来
    controller.run()
    # 数据的存储
    controller.SaveDataToCSV()
    # 避免等待，提前告知结束
    g.msgbox("代码运行结束了！", image='Warning.gif')

    # 测试完后切换手机电池为充电状态  --2 为充电状态
    os.popen("adb shell dumpsys battery set status 2")
    print("已切换回充电模式...")

    # 存入结束时间
    end = time.clock()
    print("整个测试总共花费了 %.1f 秒" % (end - start))
