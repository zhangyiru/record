背景：

iso和img都已删除，通过virsh undefine无法直接删除kvm，出现以下错误：

Requested operation is not valid: cannot undefine domain with nvram



使用--nvram解决

virsh undefine xxx --nvram

https://www.cnblogs.com/loudyten/articles/10233268.html