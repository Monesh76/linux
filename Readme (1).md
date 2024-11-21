`# KVM Exit Tracking Assignment`

``This README details the steps to reproduce the assignment for modifying the KVM code to track exit types, testing the modified KVM code by booting an inner VM, and periodically checking `printk` output in the outer VM. It also includes observations and answers to related questions.``

`---`

`## Steps to Complete the Assignment`

`### Prerequisites`  
`- A working configuration from Assignment 1.`

`---`

`### Step 1: Fork and Clone the Linux Kernel Repository`  
`1. Fork the [Linux kernel repository](https://github.com/torvalds/linux) on GitHub.`  
`2. Clone the forked repository into the outer VM:`  
   ```` ```bash ````  
   `git clone https://github.com/<your-username>/linux.git`  
   `cd linux`

Configure the kernel:  
bash  
Copy code  
`make menuconfig`

3.   
   * Enable KVM support in the kernel configuration (Virtualization options \-\> KVM for x86 processors).

Install necessary tools:  
bash  
Copy code  
`sudo apt install make build-essential libncurses-dev bison flex libssl-dev libelf-dev zstd`

4. 

Clean the build environment:  
bash  
Copy code  
`make clean`  
`make mrproper`

5. 

Copy the GCP config file:  
bash  
Copy code  
`cp /boot/<your-gcp-config> ~/linux/.config`

6. 

Clear trusted and revocation keys in `.config`:  
bash  
Copy code  
`vi .config`  
Set:  
text  
Copy code  
`CONFIG_SYSTEM_REVOCATION_KEYS=""`  
`CONFIG_SYSTEM_TRUSTED_KEYS=""`

7. 

Update configuration:  
bash  
Copy code  
`make oldconfig`

8. 

Build the kernel:  
bash  
Copy code  
`make -j$(nproc)`

9. 

Generate `initramfs`:  
bash  
Copy code  
`sudo update-initramfs -c -k $(make kernelrelease)`

10. 

Install the kernel:  
bash  
Copy code  
`sudo make modules_install`  
`sudo make install`

11. 

Update GRUB and reboot:  
bash  
Copy code  
`sudo update-grub`  
`sudo reboot`

12. 

---

### **Step 2: Verify and Modify KVM**

After reboot, verify the new kernel:  
bash  
Copy code  
`uname -r`

1. Expected output: `6.12.0+`.

Verify KVM functionality:  
bash  
Copy code  
`lsmod | grep kvm`

2. 

Edit the KVM exit handler:  
bash  
Copy code  
`cd ~/linux/arch/x86/kvm`  
`vi x86.c`

3. 

Add the following code to track and log exit types:  
c  
Copy code  
`static unsigned long long exit_counts[256] = {0};`  
`static unsigned long long total_exit_count = 0;`  
`#define EXIT_PRINT_THRESHOLD 10000`

`static const char *get_exit_reason_name(int reason) {`  
    `switch (reason) {`  
        `case 0: return "EXCEPTION_NMI";`  
        `case 1: return "EXTERNAL_INTERRUPT";`  
        `case 2: return "TRIPLE_FAULT";`  
        `case 9: return "CPUID";`  
        `case 28: return "HLT";`  
        `case 30: return "MSR_WRITE";`  
        `case 48: return "EPT_MISCONFIG";`  
        `default: return "UNKNOWN_EXIT";`  
    `}`  
`}`

`static void log_exit_statistics(void) {`  
    `int i;`  
    `printk(KERN_INFO "KVM Exit Statistics (Every %d exits):\n", EXIT_PRINT_THRESHOLD);`  
    `for (i = 0; i < 256; i++) {`  
        `if (exit_counts[i] > 0) {`  
            `printk(KERN_INFO "Exit Type: %d (%s) - Count: %llu\n",`  
                   `i, get_exit_reason_name(i), exit_counts[i]);`  
        `}`  
    `}`  
`}`

4. 

Rebuild the kernel and install:  
bash  
Copy code  
`make -j$(nproc)`  
`sudo make modules_install`  
`sudo make install`  
`sudo update-initramfs -c -k $(make kernelrelease)`  
`sudo update-grub`  
`sudo reboot`

5. 

Start the inner VM and observe exit statistics:  
bash  
Copy code  
`sudo dmesg`

6. 

---

### **Step 3: Commit Changes to GitHub**

Commit and push changes:  
bash  
Copy code  
`git add .`  
`git commit -m "Add KVM exit tracking functionality"`  
`git push origin master`

1. 

---

## **Observations**

### **Frequency of Exits**

* **General Trend**: Exit counts increase gradually during VM operation.  
* **Surge**: Certain operations (e.g., MSR\_WRITE) exhibit high exit counts.

### **Estimated Exits During VM Boot**

* \~10,000–20,000 exits are logged for every 10,000-threshold snapshot.  
* Total exits exceed 100,000 during a full VM boot.

### **Most Frequent Exit Types**

1. **Exit Type 30: MSR\_WRITE**  
   * Dominates logs, exceeding 200,000 counts.  
   * Related to virtualization control.  
2. **Exit Type 28: HLT**  
   * Common during idle states.  
3. **Exit Type 1: EXTERNAL\_INTERRUPT**  
   * Related to interrupts handled by the hypervisor.

### **Least Frequent Exit Types**

1. Exit Types 29, 46, 47, 54, 55, and 12 (UNKNOWN\_EXIT)  
   * Occur rarely, often due to specific VM states.

