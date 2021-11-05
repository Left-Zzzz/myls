# 实现带-a-l的简单ls命令

## 实现功能
1. 实现 `-l` 、 `-a` 两个选项，输出样式基本参照系统的 ls 命令
2. ⽀持传⼊多个参数
3. ⽀持缺省参数
4. 对于输出，依照⽂件名按照字典序排序
5. 输出对⽬录和可执⾏⽂件带颜⾊
6. 程序结束，成功返回0，失败返回⾮零值

------

## 代码
```c
#include<stdio.h>
#include<sys/types.h>
#include<sys/stat.h>
#include<fcntl.h>
#include<dirent.h>
#include<stdlib.h>
#include<unistd.h>
#include<string.h>
#include<time.h>
#include<pwd.h>
#include<grp.h>
//根据uid打印名字
char *uid_to_name(uid_t uid)
{
   struct passwd * getpwuid(),*pw_ptr;
   static char numstr[10];

   if((pw_ptr= getpwuid(uid))==NULL)
   {
       sprintf(numstr,"%d",uid);
       return numstr;
   }
   else
       return pw_ptr->pw_name;
}

char *gid_to_name(gid_t gid)
{
   struct group *getgrgid(),*grp_ptr;
   static char numstr[10];
   if((grp_ptr=getgrgid(gid))==NULL)
   {
       sprintf(numstr,"%d",gid);
       return numstr;
   }
   else
       return grp_ptr->gr_name;
}

void print_more(struct stat* st)
{
    int file_type = (st -> st_mode & 0170000);
    char file_type_info[11];
    //处理文件类型
    switch(file_type)
    {
        case 0140000:
            file_type_info[0] = 's';
            break;
        case 0120000:
            file_type_info[0] = 'l';
            break;
        case 0100000:
            file_type_info[0] = '-';
            break;
        case 0060000:
            file_type_info[0] = 'b';
            break;
        case 0040000:
            file_type_info[0] = 'd';
            break;
        case 0020000:
            file_type_info[0] = 'c';
            break;
        case 0010000:
            file_type_info[0] = 'f';
            break;
    }
    //处理文件权限
    file_type_info[1] = st -> st_mode & 0000400 ? 'r' : '-';
    file_type_info[2] = st -> st_mode & 0000200 ? 'w' : '-';
    file_type_info[3] = st -> st_mode & 0000100 ? 'x' : '-';
    file_type_info[4] = st -> st_mode & 0000040 ? 'r' : '-';
    file_type_info[5] = st -> st_mode & 0000020 ? 'w' : '-';
    file_type_info[6] = st -> st_mode & 0000010 ? 'x' : '-';
    file_type_info[7] = st -> st_mode & 0000004 ? 'r' : '-';
    file_type_info[8] = st -> st_mode & 0000002 ? 'w' : '-';
    file_type_info[9] = st -> st_mode & 0000001 ? 'x' : '-';
    //处理文件修改时间
    char* modify_time;
    modify_time = ctime(&st -> st_mtime);
    modify_time += 4;
    modify_time[12] = 0;
    //打印详细信息
    printf("%s %ld %s %s %8ld %s ", file_type_info, st -> st_nlink,
           uid_to_name(st -> st_uid), gid_to_name(st -> st_gid),
           st -> st_size, modify_time);
}

int find_mid(char(*data)[256], int l, int r)
{
    int mid = (l + r) / 2;
    if(strcmp(data[l], data[mid]) > 0)
    {
        int t = l;
        l = mid;
        mid = t;
    }
    if(strcmp(data[mid], data[r]) > 0)
    {
        int t = mid;
        mid = r;
        r = t;
    }
    return mid;
}

void swap(char *a, char *b)
{
    char t[256];
    strcpy(t, a);
    strcpy(a, b);
    strcpy(b, t);
}
//使用快排对文件列表按字典序排序
void sort(char(*data)[256], int x, int y)
{
    int l = x, r = y;
    while(l < r)
    {
        int mid = find_mid(data, l, r);
        do
        {
            while(strcmp(data[r], data[mid]) > 0)
                r--;
            while(strcmp(data[l], data[mid]) < 0)
                l++;
            if(l <= r)
            {
                swap(data[l++], data[r--]);
            }
        }
        while(l <= r);
        sort(data, l, y);
        l = x;
    }
}
int check_filename_color(int mode_bit)
{
    int file_type = (mode_bit & 0170000);
    if(file_type == 0040000) return 2;
    if(mode_bit & 0000111) return 1;
    return 0;
}

void printdir(char* dirname, int is_all, int is_more)
{
    char pathname[512];
    char names[256][256];
    int cnt_name = 0;
    DIR* dir;
    struct dirent* dp;
    struct stat st;
    //打开目录
    if(!(dir = opendir(dirname)))
    {
        perror("opendir");
        exit(1);
    }
    //检索目录项，根据is_all决定是否过滤.开头文件
    while(dp = readdir(dir))
    {
        if(!is_all && dp -> d_name[0] == '.')
            continue;
        strcpy(names[cnt_name++], dp -> d_name);
    }
    //对检索到的文件列表按字典序排序
    sort(names, 0, cnt_name - 1);
    //printf("[%s]\n", names[0]);
    for(int i = 0; i < cnt_name; i++)
    {
        sprintf(pathname, "%s/%s", dirname, names[i]);
        if(stat(pathname, &st) == -1)
        {
            perror("stat");
            exit(1);
        }
        //（当前可省略）如果文件为目录，递归遍历该目录下的文件
        /*
        if(S_ISDIR(st.st_mode))
        {
            if(strcmp(dp -> d_name, ".") && strcmp(dp -> d_name, ".."))
                printdir(pathname, ismore);
        }
        */
        if(is_more) print_more(&st);//显示详细信息
        //标记文件名颜色：0表示默认，1表示绿色，2表示蓝色
        int filename_color = check_filename_color(st.st_mode);
        if(filename_color == 1)
        {
            printf("\33[1;32m%s\33[0m\n", pathname + 2);
        }
        else if(filename_color == 2)
        {
            printf("\33[1;34m%s\33[0m\n", pathname + 2);
        }
        else printf("%s\n", pathname + 2);

    }
    closedir(dir);
}
int main(int argc, char* argv[])
{
    int is_all = 0, is_more = 0;
    if(argc == 1)
    {
        printdir(".", is_all, is_more);
        return 0;
    }
    if(argc > 1)
    {
        for(int i = 1; i < argc; i++)
        {
            if(argv[i][0] != '-') continue;
            for(int j = 1; argv[i][j]; j++)
            {
                if(argv[i][j] == 'a') is_all = 1;
                else if(argv[i][j] == 'l') is_more = 1;
                else
                {
                    printf("invalid option, please check again!\n");
                    exit(1);
                }
            }
        }
    }
    int flag = 0;
    for(int i = 1; i < argc; i++)
    {
        if(argv[i][0] == '-') continue;
        flag = 1;
        printdir(argv[i], is_all, is_more);
    }
    if(!flag) printdir(".", is_all, is_more);
    return 0;
}

```

------
## 重要函数介绍
1. `int main(int argc, char* argv[])`：
- 功能：对于给出的参数，检索出目录名和选项，并调用 `print_dir()` 。
- 参数：
  1.  `int is_all`：-a选项标志位
  2. `int is_more`：-l选项标志位
  3. `int flag`：目录是否缺省

2. `void printdir(char* dirname, int is_all, int is_more)`：
- 功能：按指定要求打印当前目录下的文件。
- 参数：
  1. `char* dirname`：指定目录名
  2. `int is_all`：-a选项标志位
  3.  `int is_more`：-l选项标志位
  4. `int flag`：目录是否缺省
  5. `char pathname[512]`：文件完整路径名
  6. `char names[256][256]`：记录需要打印的文件名的数组
  7. `int cnt_name = 0`：记录需要打印的文件名的数量
  8. `DIR* dir`：DIR指针，记录目录相关信息，可以用来读取目录项
  9. `struct dirent* dp`：dirent指针，储存目录项的信息
  10. `struct stat st`：记录inode的结构体

3. `void sort(char(*data)[256], int x, int y)`
- 功能：对数组中的文件名安装字典序进行排序，使用单边递归优化的快排实现。
- 参数：
  1. `char(*data)[256]`：记录文件名的数组
  2. `int x`：数组的下边界
  3.  `int y`：数组的上边界
  4. `int l` `int r` `int mid`： 每次循环遍历需要用到的左指针、右指针、哨兵指针

4. `void print_more(struct stat* st)`
- 功能：打印文件详细内容，与 `ls -l` 基本一致。
- 参数：
  1. `struct stat* st`：相当于文件的inode表
  2. `int file_type`：保存文件类型与权限信息的掩码，根据与运算结果转换成与`ls -l`内容一致的文件权限信息
  3. `char file_type_info[11]`：保存与`ls -l`内容一致文件类型与权限信息
  4. `char* modify_time`：储存文件信息最后一次被修改的时间，格式与`ls -l`一致。
  

------

## 运行截图
![在这里插入图片描述](images/1.jpg)

![在这里插入图片描述](images/2.jpg)

![在这里插入图片描述](images/3.jpg)

![在这里插入图片描述](images/4.jpg)

![在这里插入图片描述](images/5.jpg)


-------
## 笔记
1. `DIR dir` 中储存的是目录信息；通过 `DIR* opendir(char* dirname)` 可以打开文件，得到记录目录信息的结构体（目录流）；再通过 `struct dirent* readdir(DIR* dir)` 可以循环得到目录中的目录项；最后通过 `struct stat*(char *pathname, struct stat *st)` 可以得到对应文件的stat结构体，相当于inode表，通过stat结构体和dirent结构体我们就可以得出文件的有关信息。
2. stat结构体中的 `st_mode` 储存的是文件类型和文件权限的掩码，占6个八进制位。其中高2位为文件的类型信息，低4位储存文件的权限信息，文件权限信息高一位储存的是文件的特殊权限信息。