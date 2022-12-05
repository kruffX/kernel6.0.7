# kernel6.0.7
#### 下载必备依赖包：

    sudo passwd
    su
    yum -y install gcc gcc-g++ ncurses ncurses-devel
    yum -y install *curses* elfutils-libelf-devel openssl-devel dwarves
    yum -y install openssl flex bison
    yum -y install openssh-server
    iptables -A INPUT -p tcp --dport 22 -j ACCEPT
    systemctl enable sshd.service
    systemctl start sshd.service
    ip addr

通过ssh将内核包导入虚拟机并解压缩：
    
    tar zxvf Downloads/linux-6.0.7.tar.gz -C /usr/src/

进入文件夹：

    cd /usr/src/linux-6.0.7
    
文件内容修改
    
    vi include/linux/sched.h L1514
    :set nu
    int hide;
    int old_pid;
    
    vi kernel/fork.c L2088
    p->old_pid = pid;
    p->hide = 0;
    
    vi kernel/sys.c L2791
    SYSCALL_DEFINE2(hide, pid_t, pid, int, on)
    {
        struct task_struct *p;
	    struct task_struct * me = NULL;
	    p = &init_task;
	    do
	    {
		    if( pid == p->old_pid )
		    {
			    me = p;
			    break;
		    }	
        }while( ( p = next_task(p) ) && ( p != &init_task ) );
	
	    if( current->cred->uid.val != 0 || me == NULL ) //判断是否为root用户或者pid是否有效
		    return 0;
	
	    if( on == 1 )
	    {  
		    me->pid = 0;   //置pid为0，隐藏
		    me->hide = 1;  //置隐藏标志为1
	    }
	    else
	    {
		    if( me->hide == 1 )
		    {
			    me->pid = me->old_pid;  //取消隐藏
			    me->hide = 0;
		    }
	    }
	    return 0;
    }
    SYSCALL_DEFINE2(hide_user_process, uid_t, uid, char*, binname)
    {
	    if( current->cred->uid.val != 0 )
		    return 0;
	
	    struct task_struct *p;
	    p = &init_task;
	    if( binname == NULL )  //隐藏该用户所有进程
	    {
		    do
		    {
			    if( p->cred->uid.val == uid )
			    {
				    p->pid = 0;
				    p->hide = 1;
			    }
		    }while( ( p = next_task(p) ) && ( p != &init_task ) );
        }
	    else  //隐藏特定进程
	    {
		    do
		    {
			    if( p->cred->uid.val == uid )
			    {
			        int flag = 1;
				    int i = 0;
				    for( i = 0; ( binname[ i ] != NULL && p->comm[ i ] != NULL ); i++ )
				    {
					    if( binname[ i ] != p->comm[ i ] )
					    {
						    flag = 0;
						    break;
					    }
				    }
				    if( flag == 1 && binname[ i ] == NULL && p->comm[ i ] == NULL )  //判断名字是否相同
				    {
					    p->pid = 0;
					    p->hide = 1;
				    }
			    }
		    }while( ( p = next_task(p) ) && ( p != &init_task ) );
	    }

        return 0;
    }
   
    vi include/uapi/asm-generic/unistd.h L887
    #define __NR_hide 451
    #define __NR_hide_user_process 452
    // 修改：
    #define __NR_syscalls 453
    
    vi ../../include/asm-generic/unistd.h L887
    #define __NR_hide 451
    #define __NR_hide_user_process 452
    // 修改：
    #define __NR_syscalls 453
    
    vi arch/x86/entry/syscalls/syscall_64.tbl L375
    451	common	hide			sys_hide
    452	common	hide_user_process	sys_hide_user_process
    
内核编译

    make mrproper
    cp /boot/config-6.0.7-301.fc37.x86_64 .config
    make menuconfig
    make all
    make modules_install
    make install
    reboot

测试程序
	
	// test1.c
	#include<linux/unistd.h>
	#include<sys/syscall.h>
	#include<stdio.h>
	#define __NR_hide_user_process 452
	void main()
	{
		printf( "%d", 6 );
		syscall(__NR_hide_user_process, 1000, 0);  //uid 1000是本机用户，uid 0 是//root用户
		printf( "%d",  6);
	}
	
	//test2.c
	#include<linux/unistd.h>
	#include<sys/syscall.h>
	#include<stdio.h>
	#define __NR_hide 451
	void main()
	{
		printf("%d", 5);
		syscall(__NR_hide, 2476, 1);
		printf("%d", 5);
	}
