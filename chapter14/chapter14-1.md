

## Node.js在物联网

### 1、Build Node.js for Android
#### Linux构建环境
```shell
jiangling@young:~/node/deps/npm$ uname -a
Linux young 3.11.0-15-generic #25~precise1-Ubuntu SMP Thu Jan 30 17:39:31 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
```
#### 32位 Android NDK
```shell
drwxr-xr-x 10 jiangling jiangling      4096  3月  1  2014 android-ndk-r9d
```

#### clone node.js
```shell
git clone https://github.com/joyent/node.git
```

#### android-configure patch
```shell
jiangling@young:~/node$ git diff
diff --git a/android-configure b/android-configure
index 7acb7f3..aae0bf1 100755
--- a/android-configure
+++ b/android-configure
@@ -3,7 +3,7 @@
 export TOOLCHAIN=$PWD/android-toolchain
 mkdir -p $TOOLCHAIN
 $1/build/tools/make-standalone-toolchain.sh \
-    --toolchain=arm-linux-androideabi-4.7 \
+    --toolchain=arm-linux-androideabi-4.8 \
     --arch=arm \
     --install-dir=$TOOLCHAIN \
     --platform=android-9
```
否则会出现`arm-linux-androideabi-gcc not found`的错误。
#### configure && make
```shell
source ./android-configure ~/android-ndk-r9b
mv python2.7 oldpython2.7
ln -s /usr/bin/python2.7 python2.7
cd  ~/node
make
```
node bin大小
```shell
jiangling@young:~/node$ file out/Release/node 
out/Release/node: ELF 32-bit LSB executable, ARM, version 1 (SYSV), dynamically linked (uses shared libs), not stripped

root@android:/data/local/tmp # ls -al node
ls -al node
-rwx------ shell    shell    12158228 2014-11-24 17:01 node
```
配置without-ssl后大小
```
jiangling@young:~/node$ ls -al out/Release/node 
-rwxrwxr-x 1 jiangling jiangling 9804644 11月 26 13:55 out/Release/node
```

### 2、Run Node.js on Android
这边我选择了ROOT过的小米M1手机， 安装了”瑞士军刀“[busybox](http://www.busybox.net/downloads/binaries/latest/
), 以便查看系统信息。
```shell
root@android:/data/local/tmp # ./busybox uname -a
./busybox uname -a
Linux localhost 3.4.0-perf-g1ccebb5-00146-gd6845ec #1 SMP PREEMPT Mon Nov 4 20:10:00 CST 2013 armv7l GNU/Linux
```

用ADB将 `node`, `test.js`(经典的hello world) push到M1上：
```shell
adb push d:\node /data/local/tmp
adb push d:\test.js /data/local/tmp
adb shell chmod 755 /data/local/tmp/node
```

运行 和 结果
![360_1126_10_38_01](http://img1.tbcdn.cn/L1/461/1/c60b8459e72b1c8f2356f6482d09cec2194d6abe)
![360_1125_16_29_01](http://img3.tbcdn.cn/L1/461/1/219c69b83c49b3ff789190c3378ac017c148e240)






