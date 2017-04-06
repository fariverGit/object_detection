### python调用darknet中的函数
#### 首先需要把darknet源码制作为一个.so文件
直接编写Makefile或者用cmake都可以实现这个功能
##### Makefile
Makefile是作者代码中原生提供的，但是只能编译为可执行文件。我在原生Makefile中的改动：
```
CFLAGS=-Wall -Wfatal-errors -fPIC
$(CC) $(COMMON) $(CFLAGS) -shared $^ -o $@ $(LDFLAGS)
```
**注意：**
- CFLAGS中不要加-g参数，否则会去找cvRound等用不着的老版opencv函数。至于如果debug，我还在探索中...
- 生成库之后先用file命令查看一下是否是 shared object
- 同样，如何需要生成可执行文件也用file查看一下是否是 executable, 因为linux不是以后缀判断文件类型的。如果非要./shared-object的话，连main函数都进入不了就会segmentation fault。
##### cmake
因为Makefile的代码比较多，而且内容不易懂，所以我中间尝试选择cmake来编译动态库。贴上CMakeLists.txt:
```
CMAKE_MINIMUM_REQUIRED(VERSION 2.8.3)
set(ProjectName libdarknet.so)

file(GLOB_RECURSE c_files "${CMAKE_CURRENT_SOURCE_DIR}/src/*.c")
file(GLOB_RECURSE cu_files "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cu")
file(GLOB_RECURSE head_files "${CMAKE_CURRENT_SOURCE_DIR}/src/*.h")

#find opencv lib
find_package(OpenCV 3 REQUIRED)

#message("cmake:USE_xxxlib on")
include_directories(head_files)
        
#check out compiler type and add compiler option
if(${CMAKE_COMPILER_IS_GNUCC})
        message (STATUS "add -Wall -fPIC -DGPU -DCUDNN in cxx_flags")
	        #add_definitions(-Wall -fPIC -DOPENCV -DGPU -DCUDNN)
            add_definitions(-Wall -fPIC -DGPU -DCUDNN)
        endif()


#add all *.cpp source code together and compile them to object
add_executable(${ProjectName} ${head_files} ${c_files}) 
cuda_add_library(cu_object SHARED ${head_files} ${cu_files}) 

#link libraries to excutable file
set(link_flags -lm -pthread -shared -L/usr/local/cuda/lib64 -lcuda -lcudart -lcudnn -lcublas -lcurand)
target_link_libraries(${ProjectName} ${OpenCV_LIBS} cu_object ${link_flags})
```
比较奇怪的是cmake版的libdarknet.so在使用过程中 ，会去找cvRound等用不着的老版opencv函数。即库中存在不需要的符号链接，但如果不把opencv编译进来就可以正常使用(darknet不编译opencv也可以正常使用)。
