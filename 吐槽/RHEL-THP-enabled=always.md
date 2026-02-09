# 问题
RHEL默认将THP-enabled设置为always，会对终端用户造成困扰。
当用户发现内存不足的问题的时候，才能发现是这个设置有问题，需要修改成never。
# 建议
将这个值修改为madvise

# 相同疑问
https://access.redhat.com/discussions/85827302-b3d8-4dbd-b149-81ba97b7eb3c