
# Adding a sys call

Documenting the process of adding a system call do my debian kernel.
Following the guide provided by my professor for our homework.

1. Finding and booting known good kernel.
   1. Debian version 6.1.0-17-amd64
   2. Dont do anything to file vmlinux-6.1.0-17-amd64
2. Compiling own working kernel
   1. `sudo apt install build-essential pahole libelf-dev libncurses-dev libssl-dev flex bison` ran this to get everything required for compile
   2. `sudo apt install linux-source` installed linux source
   3. linux 6.1 source decompressed into home dir. It's 1.5 G
   4. renamed directory to /pa2
   5. ![ls pa2](https://github.com/redassser/syscall-adding/assets/40395425/bba50ed1-566a-45c0-9899-6942242141e6)

   6. config copied from known good kernel
   7. Changed localversion

   ```
   < CONFIG_LOCALVERSION=""
   ---
   \> CONFIG_LOCALVERSION="-rpiedrah-pa2"
   32a33
   ```

   8. Time to compile.
      1. Started at 6:55 PM 2/9/2024
      2. Paused for like 20 minutes.
      3. Finished around 9 PM
   9. Size after compile is 19G
   10. Installed modules ![du s h modules](https://github.com/redassser/syscall-adding/assets/40395425/6bc0af16-e9ba-4426-8e7c-6ebd1614906b)

   11. Time to install.
   12. It boots!
   13. ![uname kernel](https://github.com/redassser/syscall-adding/assets/40395425/2342eb57-bfb3-41bb-b377-836aa2f11ba0)

After this I made a new backup copy of the kernel in another directory.
3. Creating a kernel module
   1. Made a file called rpiedrah.c that will hold the new module

```C
#include <linux/module.h>
#include <linux/init.h>
#include <linux/sched.h>
#include <asm/current.h>
static int init_r(void) {
    printk(KERN_ALERT "Hello World from RPIEDRAH\n");
    return 0;
}
static void exit_r(void) {
    printk(KERN_ALERT "PID is %i and program name is %s\n", current->pid, current->comm);
}
module_init(init_r);
module_exit(exit_r);
MODULE_LICENSE("Dual BSD/GPL");
```

WORKING!!!

![module loaded](https://github.com/redassser/syscall-adding/assets/40395425/e6e12664-a5f1-41fc-80f2-8716cf5542d0)


4. Adding a System Call
   1. edited pa2/arch/x86/entry/syscalls/syscall_64.tbl or whatever to have the new syscall
      1. `451 common rpiedrah_syscall sys_rpiedrah_syscall`
   2. edited pa2/include/linux/syscalls.h to include new syscall line 1060 - 1062

   ```h
   //RPIEDRAH custom system call

   asmlinkage long sys_rpiedrah_syscall(char __user *st);
   ```

   3. edited pa2/Makefile to include my_syscall folder to make
      1. `core-y  := my_syscall/`

Using this to test initially

```C
#include <linux/syscalls.h>
#include <string.h>
SYSCALL_DEFINE1(rpiedrah_syscall, char __user *, st) {
   size_t leng = strlen(st);
   char *stk;

   stk = kmalloc(leng + 1, GFP_KERNEL);
   strcpy(stk, st);
   printk(KERN_ALERT "%s",stk);
}
```

The kernel took about an hour and a half to compile.
After installing the new kernel I could confirm that the syscall was present

Code for testing the system call!

```C
#define _GNU_SOURCE
#include <unistd.h>
#include <sys/syscalls.h>
#define STDOUT 1
int main(int argc, char **argv) {
   char msg[] = "Hi there\n";
   syscall(451, msg);
   return 0;
}
```

IT FUCKING WORKED!!!!!!!!!!!!!!!!!!!!!!!!!!!!
![testing worked!!!!](https://github.com/redassser/syscall-adding/assets/40395425/23dd6944-80fd-4086-addb-b319fbd28952)


Now for the real syscall code.

```C
#include <linux/syscalls.h>
#include <string.h>
SYSCALL_DEFINE1(rpiedrah_syscall, char __user *, st) {
   size_t leng = strlen(st);
   int i;
   int r = 0;
   char c;
   char *stk;

   if(st == NULL) {
      return -1;
   } else if (leng >= 32)
      return -1;
   stk = kmalloc(leng + 1, GFP_KERNEL);
   if(copy_from_user(stk, st, leng))
      return -EFAULT;
   printk(KERN_ALERT "before: %s\n", stk);

   for(i = 0; i < leng; i++) {
      c = stk[i];
      if (c=='a'||c=='e'||c=='i'||c=='o'||c=='u'||c=='y') {
         stk[i] = 'R'; r++;
      }
   }
   printk(KERN_ALERT "after: %s\n", stk);
   if(copy_to_user(st, stk, leng))
      return -EFAULT;
   return r;
}
```

after this its time to recompile. Took about 5 minutes

works as intended ![final capture uname](https://github.com/redassser/syscall-adding/assets/40395425/74325e2a-6e3b-4266-bb19-cceecaae3fa5)




ALMOST forgot my user code

```C
#define _GNU_SOURCE
#include <unistd.h>
#include <stdio.h>
#include <sys/syscall.h>
#define STDOUT 1
int main(int argc, char **argv) {
	char msg[] = "What is the real message?";
	char msg2[] = "This string is too long to fit in 32 chars so it will not get through";
	long res = 0; long res2 = 0;
	res = syscall(451, &msg);
	res2 = syscall(451, &msg2);
	printf("Return value of short msg: %i\n", res);
	printf("Return value of long msg: %i\n", res2);
	return 0;
}

```
