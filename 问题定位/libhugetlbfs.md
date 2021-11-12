direct (2M: 64):	Bad configuration: Failed to open direct-IO file: Invalid argument



dfd = open(TMPFILE, O_CREAT|O_EXCL|O_DIRECT|O_RDWR, 0600);



![img](file:///C:/Users/z00585918/AppData/Roaming/eSpace_Desktop/UserData/z00585918/imagefiles/3735730C-CB66-45EC-A1FC-D8DF17CC1AE2.png)
问下，libhugetlbfs使能make check，有个direct.c的用例挂在了这个open，open的手册说是O_DIRECT不支持在tmpfs上，那这个我是提个代码把用例代码改了？
![img](file:///C:/Users/z00585918/AppData/Roaming/eSpace_Desktop/UserData/z00585918/imagefiles/D3A9642E-4E66-45B0-B17D-AB0F53E4941D.png)
![img](file:///C:/Users/z00585918/AppData/Roaming/eSpace_Desktop/UserData/z00585918/imagefiles/269A6588-F847-4924-BD85-3B4AECE16239.png)