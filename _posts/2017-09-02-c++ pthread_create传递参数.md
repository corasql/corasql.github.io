# c++ pthread_create传递参数

## 传递一个值

```
#include <iostream>
#include <pthread.h>
using namespace std;
pthread_t thread;
void *fn(void *arg)
{
    int i = *(int *)arg;
    cout<<"i = "<<i<<endl;
    return ((void *)0);
}
int main()
{
    int err1;
    int i=10;
    err1 = pthread_create(&thread, NULL, &fn, &i);
    pthread_join(thread, NULL);

}
```


## 不同线程传递不同值（错误方式）

```
#include <iostream> 
#include <pthread.h>
using namespace std;
pthread_t thread[5];

void *fn(void *arg)
{
    int i = *(int *)arg;
    cout<<"i = "<<i<<endl;

    return ((void *)0);
}

int main()
{
    int err1;
    int i=0;
    for(;i<5;i++)
    {
         err1 = pthread_create(&thread[i], NULL, &fn, &i);
    }
    for(i=0;i<5;i++){
       pthread_join(thread[i], NULL);
    }

}
[aboss@aboss5 svn_test]$ ./pthread_create_para2 
i = 1
i = 2
i = 4
i = 2
i = 2
```
本来是想实现每创建一个线程，传递一个不同的i值，而结果却不是想要的，原因是由于共用i变量导致

## 不同线程传递不同值（正确方式）

```
#include < iostream>
#include < pthread.h>
using namespace std;
pthread_t thread[5];
void *fn(void *arg)
{
    int i = *(int *)arg;
    cout<<"i = "<<i<<endl;
    return ((void *)0);
}
int main()
{
    int err1;
    int i=0;
    int j[5];
    for(;i<5;i++)
    {
        j[i]=i;
         err1 = pthread_create(&thread[i], NULL, &fn, &j[i]);
    }
    for(i=0;i<5;i++){
       pthread_join(thread[i], NULL);

    }
}
```

## 通过结构体来传递多个值

```
#include <iostream>
#include <unistd.h>
#include <pthread.h>
using namespace std;

struct mypara{
    int para1;
    double para2;
};

void *fn(void *arg)
{
    mypara p=*(mypara*)arg;
    cout<<p.para1<<endl;
    cout<<p.para2<<endl;
    return NULL;
}

int main()
{
    int err1;
    int i=0;
    pthread_t thread[5];
    mypara para[5];
    for(;i<5;i++)
    {
      para[i].para1=i;
      para[i].para2=9.01;
      err1 = pthread_create(&thread[i], NULL, &fn, &para[i]);
    }
    for(i=0;i<5;i++){
        pthread_join(thread[i],NULL);
    }
    return 0;
}

```