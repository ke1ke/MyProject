# MyProject
document my code
//项目----文件传输----
//一、需求：
//1.c/s模式，客户端至少可以支持两个
//2.客户端与服务器端有交互能力。
//a.客户端可查看服务端文件。
//b.服务器端删除文件
//c.拷贝、新建、移动、重命名服务端文件（普通文件，目录文件）
//d.多个客户端可同时下载上传文件
//二、思路：
//1.先创建完整的套接字让服务器与客户端可以建立连接。
//2.用多线程去实现对多个用户的管理。
//3.规定协议：以原来命令的名称通知服务端（ls、pwd、rm）。
//****************客户端文件*******************
//=================cli.c=================
# include <stdio.h>
# include <assert.h>
# include <stdlib.h>
# include <string.h>
# include <unistd.h>
# include <sys/socket.h>
# include <arpa/inet.h>
# include <netinet/in.h>
# include <fcntl.h>

int main()
{
    int sockfd = socket(AF_INET,SOCK_STREAM,0);
    assert(sockfd != -1);

    struct sockaddr_in saddr;
    memset(&saddr,0,sizeof(saddr));
    saddr.sin_family = AF_INET;
    saddr.sin_port = htons(6000);
    saddr.sin_addr.s_addr = inet_addr("127.0.0.1");
    int res = connect(sockfd,(struct sockaddr*)&saddr,sizeof(saddr));
    assert(res != -1);

    while(1)
    {
        char buff[128] = {0};
    
        printf("connect [yonghu~ ]$");
        fflush(stdout);
        fgets(buff,128,stdin);
        buff[strlen(buff)-1] = 0;
        if(strcmp(buff,"end,3")==0)
        {
            break;
        }
        char tmp[128] = {0};
        strcpy(tmp,buff);
        char *myargv[10] = {0};
        char *s = strtok(tmp," ");
        int i = 0;
        while(s != NULL)
        {
            myargv[i++] = s;
            s = strtok(NULL," ");
        }
        if(myargv[0] == NULL)
        {
            continue;
        }
        if(strcmp(myargv[0],"end")==0)
        {
            break;
        }
        if(strcmp(myargv[0],"get")==0)
        {
            if(myargv[1]==NULL)
            {
                printf("no filename!\n");
                continue;
            }
            send(sockfd,buff,strlen(buff),0);
            recv_file(sockfd,myargv[1]);
        }
        else if(strcmp(myargv[0],"put")==0)
        {
            if(myargv[1] == NULL)
            {
                printf("no filename!\n");
                continue;
            }
            send(sockfd,buff,strlen(buff),0);
            send_file(sockfd,myargv[1]);
        }
        else
        {
            send(sockfd,buff,strlen(buff),0);
            char readbuff[1024] = {0};
            if(recv(sockfd,readbuff,1023,0) <= 0)
            {
                printf("ser exit!\n");
                break;
            }
            printf("%s\n",readbuff+3);
        }
    }
    close(sockfd);
}
//*******************服务器端文件*******************
//=================ser.c=================
# include <stdio.h>
# include <stdlib.h>
# include <string.h>
# include <assert.h>
# include <unistd.h>
# include <sys/socket.h>
# include <netinet/in.h>
# include <arpa/inet.h>
# include "thread.h"

# define LIS_MAX 5
int create_socket()
{
    int sockfd = socket(AF_INET,SOCK_STREAM,0);
    if(sockfd == -1)
    {
        return -1;
    }

    struct sockaddr_in saddr;//创建流式套接字
    memset(&saddr,0,sizeof(saddr));
    saddr.sin_family = AF_INET;
    saddr.sin_port = htons(6000);//端口为6000
    saddr.sin_addr.s_addr = inet_addr("127.0.0.1");//IP地址为本地主机

    int res = bind(sockfd,(struct sockaddr*)&saddr,sizeof(saddr));//绑定
    if(res == -1)
    {
        return -1;
    }

    listen(sockfd,LIS_MAX);//监听套接字
    return sockfd;
}

int main()
{
    int sockfd = create_socket();
    assert(sockfd != -1);

    while(1)
    {
        struct sockaddr_in caddr;
        int len = sizeof(caddr);
        int c = accept(sockfd,(struct sockaddr*)&caddr,&len);//接收信息
        if(c<0)
        {
            continue;
        }
        printf("accept c =%d\n",c);
        thread_start(c);//启动一个线程
    }
}
//=================thread.h=================
# include <stdio.h>
# include <stdlib.h>
# include <assert.h>
# include <string.h>
# include <unistd.h>
# include <sys/socket.h>
# include <netinet/in.h>
# include <arpa/inet.h>
# include <pthread.h>
# include <fcntl.h>

void thread_start(int c);

//=================thread.c======================
# include "thread.h"
# define MAX_ARGS 10

void* work_thread(void* arg)
{
    int c = (int)arg;
    while(1)
    {
        char buff[128] = {0};
        if(recv(c,buff,127,0)<=0)
        {
            break;
        }
        printf("buff(%d)=%s\n",c,buff);
        char *myargv[MAX_ARGS] = {0};
        char *ptr = NULL;
        char *s = strtok_r(buff," ",&ptr);
        int i = 0;
        while(s != NULL)
        {
            myargv[i++] = s;
            s = strtok_r(NULL," ",&ptr);
        }
        if(myargv[0] == NULL)
        {
            send(c,"ok#No cmd!",10,0);
            continue;
        }
        if(strcmp(myargv[0],"get") == 0)
        {
            //
        }
        else if(strcmp(myargv[0],"put")==0)
        {
            //
        }
        else
        {
            int pipefd[2];//创建管道
            pipe(pipefd);
            pid_t pid = fork();//创建进程
            assert(pid != -1);
            if(pid ==0)//子进程
            {
                dup2(pipefd[1],1);//管道写端覆盖标准输出
                dup2(pipefd[1],2);//管道写端覆盖标准错误输出
                execvp(myargv[0],myargv);//替换子进程为命令
                perror("execvp error");
                exit(0);
            }
            close(pipefd[1]);//管道关闭写端，为了从管道读端读取数据
            wait(NULL);//结束子进程，防止僵死进程
            char readbuff[1024] = {"ok#"};//不管有没有从读端读到数据都给对方发确认信息
            read(pipefd[0],readbuff+3,1020);//把读出的数据拼接到ok#后

            send(c,readbuff,strlen(readbuff),0);//把数据通过套接字发送给客户端
        }
    }
    printf("one client over\n");
}

void thread_start(int c)
{
        pthread_t id;
        pthread_create(&id,NULL,work_thread,(void*)c);
}

```
