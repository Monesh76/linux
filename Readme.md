Describe in detail the steps you used to complete the assignment. Consider your reader to be someone
skilled in software development but otherwise unfamiliar with the assignment. Good answers to this
question will be recipes that someone can follow to reproduce your development steps.

To Complete this assignment we should have a working configuration we have done in assignment 1.

STEP 1:
First we should  Fork the Linux kernel repository from GitHub i.e https://github.com/torvalds/linux

We should clone forked repository into outer vm using the below command
git clone https://github.com/<your-username>/linux.git (keep your github username in place of your-username) 
Use cd linux to change the directory

Run the make menuconfig command to configure the kernel and make sure KVM support is enabled in the kernel 
configuration. You can do this by navigating to Virtualization options and enabling KVM for x86 processors.
If the "make" is not installed you can do this by running "sudo apt install make" command
Build the Kernel with "make -j$(nproc)" Command. If you run into any errors during the build you can follow these steps

Sometimes, residual files from previous build attempts cause conflicts. Clean up the build environment with:
make clean
make mrproper

cp /boot/(your gcp config) ~/linux/.config 
Replace (your gcp config) with the actual filename of the configuration file in the /boot directory. It'll look like this
config-5.15.0-1036-gcp

Double-check that all required packages for building the kernel are installed. Run the following command to install essential tools:
sudo apt update
sudo apt install build-essential libncurses-dev bison flex libssl-dev libelf-dev

Edit the Configuration File to Clear Trusted and Revocation Keys
vi .config
Set CONFIG_SYSTEM_REVOCATION_KEYS and CONFIG_SYSTEM_TRUSTED_KEYS to Empty

CONFIG_SYSTEM_REVOCATION_KEYS=""
CONFIG_SYSTEM_TRUSTED_KEYS="" 

Install zst using "sudo apt install zstd" command
Zstandard is becoming widely used in package formats and system files because of its efficient handling of large files, especially
in newer Linux distributions and packaging systems.

Run "make oldconfig" to Update the Configuration and accept the prompts
During the make oldconfig process, you’re being prompted to confirm or select values for various kernel configuration options, 
some of which are new or have warnings about invalid settings. This process ensures that your .config file is updated to be compatible 
with the kernel source you’re building.

Now build the kernel using "make -j$nproc"

After building the kernel, you can generate the initrd/initramfs with:
sudo update-initramfs -c -k $(make kernelrelease)
Snapshot the VM: This step is a precaution. If your new kernel has issues or fails to boot, a snapshot will let you roll back the VM to 
its previous working state.

Install the Test Kernel: After verifying that everything is in place (including a snapshot for safety), you can now install the kernel with:
sudo make modules_install
sudo make install

On systems using GRUB as the bootloader, update the GRUB configuration to include the new kernel:
sudo update-grub

Reboot the system to start using the newly built kernel:
sudo reboot

STEP 2:
After the reboot, you can confirm that the new kernel is in use by running:
uname -r
This should output 6.12.0+, indicating that the system has booted into the new kernel we built.

Since the assignment involves working with KVM, check that KVM is enabled and functioning with below command
lsmod | grep kvm
This should show output related to kvm (e.g., kvm_intel or kvm_amd), indicating that KVM modules are loaded.

Locate and edit the KVM exit handler in the Linux kernel source (usually in arch/x86/kvm/ or a similar directory).
cd ~/linux/arch/x86/kvm
vi x86.c
Inside x86.c, look for the function handling VM exits i.e __vmx_handle_exit. 
This function is where KVM processes different types of VM exits

Check for __vmx_handle_exit function and change the add the code menioned below

    /* Define global counters at the top of the file */
    static unsigned long long exit_counts[256] = {0};  // Array for per-exit-type counters
    static unsigned long long total_exit_count = 0;    // Global total exit counter
    #define EXIT_PRINT_THRESHOLD 10000                 // Threshold for printing exit stats

    /* Helper function to convert exit reason to a human-readable string */
    static const char *get_exit_reason_name(int reason) {
    switch (reason) {
    case 0: return "EXCEPTION_NMI";
    case 1: return "EXTERNAL_INTERRUPT";
    case 2: return "TRIPLE_FAULT";
    case 9: return "CPUID";
    case 28: return "HLT";
    case 30: return "MSR_WRITE";
    case 48: return "EPT_MISCONFIG";
    default: return "UNKNOWN_EXIT";
    }
    }

    /* Function to log exit statistics */
    static void log_exit_statistics(void) {
    int i;
    printk(KERN_INFO "KVM Exit Statistics (Every %d exits):\n", EXIT_PRINT_THRESHOLD);
    for (i = 0; i < 256; i++) {
        if (exit_counts[i] > 0) {
            printk(KERN_INFO "Exit Type: %d (%s) - Count: %llu\n",
                   i, get_exit_reason_name(i), exit_counts[i]);
        }
    }
    }

    /* Modify the __vmx_handle_exit function */
    static int __vmx_handle_exit(struct kvm_vcpu *vcpu, fastpath_t exit_fastpath) {
    struct vcpu_vmx *vmx = to_vmx(vcpu);
    union vmx_exit_reason exit_reason = vmx->exit_reason;
    u32 vectoring_info = vmx->idt_vectoring_info;
    u16 exit_handler_index;

    /* Update exit counters */
    int exit_type = exit_reason.basic;  // Get the basic exit type
    exit_counts[exit_type]++;
    total_exit_count++;

    /* Log exit statistics every EXIT_PRINT_THRESHOLD exits */
    if (total_exit_count % EXIT_PRINT_THRESHOLD == 0) {
        log_exit_statistics();
    }

    /* Continue with the original exit handling logic */
    ...
}

Save the file and exit vi

Rebuild the kernel with your changes:
make -j$(nproc)

After kernel is ready install the kernel with below commands
sudo make modules_install
sudo make install

Regenerate the initramfs and update GRUB if necessary, then reboot the system to apply these changes:
sudo update-initramfs -c -k $(make kernelrelease)
sudo update-grub
sudo reboot
Once the system reboots, start the inner VM.

You should periodically check the printk output via “dmesg” in the outer VM to see what type(s) of exits are occurring
sudo dmesg

Commit your changes to GitHub:
git add .
git commit -m "Add KVM exit tracking functionality"
git push origin master

Question 3) Comment on the frequency of exits – does the number of exits increase at a stable rate? Or are there
more exits performed during certain VM operations? Approximately how many exits does a full VM
boot entail?
Observations: 
1)Throughout VM operation, the quantity of KVM exits increases gradually.
2)Exit counts surge for certain operations. 
MSR_WRITE (Exit Type: 30), for instance, continuously displays an extremely high count. 
The frequent MSR (Model-Specific Register) writes that occur during VM operation most likely correlate to this.
HLT (Exit Type: 28), which is probably related to halts during VM idle states, seems to occur less frequently than MSR_WRITE but is still substantial.
Although they occur far less frequently than MSR_WRITE, exits for EXTERNAL_INTERRUPT (Exit Type: 1) and other interrupt-related causes are equally noteworthy.

Estimated Exits During VM Boot: 
As seen with MSR_WRITE and other types, booting the virtual machine causes roughly 10,000–20,000 exits for every 10,000-threshold snapshot.
This suggests that there will probably be more than 100,000 exits during a full boot.

Question 4) Of the exit types, which are the most frequent? Least?

Most Frequent Exit Types:
Exit Type 30: MSR_WRITE
This exit type dominates the logs, often exceeding 200,000 counts.
Reason: MSR writes are a core part of VM operation, related to virtualization control.

Exit Type 28: HLT
Frequently seen, ranging from 10,000 to 30,000 counts, tied to halts when the VM is idle.

Exit Type 1: EXTERNAL_INTERRUPT
Appears consistently, with counts between 10,000 and 20,000. This corresponds to interrupts handled by the hypervisor.

Least Frequent Exit Types:
Exit Type 29, 47, 46: UNKNOWN_EXIT
These exit types show minimal counts (2–6 occurrences), likely due to rare or unimplemented VM instructions.

Exit Type 54 and 55: UNKNOWN_EXIT
These types also show minimal counts (around 3 occurrences) across logs.

Exit Type 12: UNKNOWN_EXIT
Counts remain very low, often below 10,000, linked to specific operations or VM state changes.
