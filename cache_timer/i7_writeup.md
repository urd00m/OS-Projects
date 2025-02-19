# I7-1068NG7 Cache Data
This is a 10th generation Intel I7 chip. 

## General Information 
Information according to Intel's datasheet for 10th generation processors. And CPUID informatino about cache and tlb information. Documentation here: https://c9x.me/x86/html/file_module_x86_id_45.html and here https://www.felixcloutier.com/x86/cpuid#tbl-3-12. 

L1 cache: 42KB (48KB reading CPUID) data cache, 32KB instruction cache, 64 byte cache line, 8-way  
L2 cache: 512KB total, 64 byte lines, 8-way  
L3 cache: 8 MB shared (2 MB per core times 4 cores), 64 byte lines, 16-way  

#### CPUID return information (eax=2, leaf 2):   
EAX - 0x 00 fe ff 01  
EBX - 0x 00 00 00 f0   
ECX - 0x 00 00 00 00   
EDX - 0x 00 00 00 00  
f0 = 64 byte prefetching  
f2 = use cpuid leaf 18h to query tlb parameters  
ff = cpuid doesn't support information use leaf 4h to query cache parameters  

#### CPUID return information (eax=4, leaf 4): 
ecx=0, the data cache (L1d):  
ways: 12  
partitions: 1  
line size: 64 bytes  
sets: 64  
total size: 48KB  

ecx=1, the instruction cache (l1i):  
ways: 8  
partitions: 1  
line size: 64 bytes  
sets: 64  
total size: 32KB  

ecx=2, the L2 cache (unified cache):  
ways: 8  
partitions: 1  
line size: 64  
sets: 1024  
total size: 512KB  

ecx=3, the L3 cache (unified cache):  
ways: 16  
partitions: 1  
line size: 64  
sets: 8192  
total size: 8MB  

#### CPUID return information (eax=0x18, leaf 18) 
ecx=0 reports eax=0x7, meaning the max ecx value we can have is 7.  

ecx=1  
Exists TLB that supports 4k, 2m, and 4m  
8-way (fully associative) 1-set TLB, Instruction TLB (level 1)  

ecx=2  
Exists TLB that supports 4k, 2m, 4m, and 1g  
16-way (fully associative) 1-set TLB, Unknown type TLB (level 1) (type id was 5, but only goes from 0-3 others are reserved so unknown what this means)  
Looking at Intel's manuals, this is maybe an instruction TLB for larger pages  

ecx=3  
Exists TLB that supports 4k  
4-way 16-set TLB, unknown type (level 1) (type id was 4)  
Looking at Intel's manuals, this is a instruction or data TLB (probably data) for smaller pages.  

ecx=4  
Exists TLB that supports 2m and 4m  
4-way 8-set TLB, unknown type (level 1) (type id was 4)  
Looking at Intel's manuals, this is a data TLB for larger pages  

ecx=5  
Exists TLB that supports 1g  
8-way (fully associative) 1-set TLB, Unknown type TLB (level 1) (type id was 4)  
Can't find a matching TLB in intel's manuals, most likely a new TLB to handle larger pages  

ecx=6  
Exists TLB that supports 4k, 2m, and 4m  
8-way 128-set TLB, Unified TLB (level 2)  

ecx=7  
Exists TLB that supports 4k and 1g  
8-way 128-set TLB, Unified TLB (level 2)  

# Forming Cache Eviction Set 
In the PACMAN paper ISCA'22 the formula used to measure cache was,  
addrs[i] = i + i * stride + i * 64 bytes (line size), where i is 1...N  

Interesting, for int[], it seems to map each int to an different pointer differing by 1, I would think they differ by 4. Since an int is 4 bytes. (Wait I think this is due to compiler or underlying language where they automatically add the *4 for you but it appears as if it is accessing 1 by 1, testing shows that they allocated ~~8 bytes for each int :\~~ 4 bytes correctly which is automatically added when doing the pointer manipulation)

#### Logs before 8/7/22 lost :(

#### Log 8/7/22
After nuking the tlbs the data generated makes no sense. Running with a stride of 16KB didn't yield any results that made any sense. The access times got shorter with more addresses being brought into cache, which makes no sense even with large N values the tlb's seemed to be able to handle it faster. After thinking, maybe it was because the TLBs could handle up to 4mb somtimes 1 gb page sizes, so maybe we need to increase the stride to 4 MB, after doing that no difference in reuslts either TLB access still got faster, not slower as N increased for some reason. 

#### Log 8/13/22 
It appears that the issues with evict + reload was caused by mfence...lfence instructions getting in the way of things causing the additional overhead. Accuracy of the evict + reload attack to determine if the secret address was accessed is around ~80%.  

Got evict + reload working on a secret. But got weird results. Using a eviction set stride of 1 gb (for some reason **.25 B or 256 MB** works equally well, but there are no page sizes that are that size so that makes no sense. Now for secret strides (strides used for the shared array) obvisouly it can't a stride less than the page size of 4 MB but the same problem from the eviction set stride **persists**. As in a secret stride of only **.25 gb or 256 MB** works. Still unknown why this happens. But it is able to determine the secret address accessed! Though further work is needed to determine why the above values are used. My only guess is that there needs to be sufficient distance from some page size bondary??? Or that the TLB needs a sufficiently large size to map things differently??? these are all guesses right now.  

Another weird thing is that for some reason in the `evict_reload.c` in order to get it reading correctly, we need to add in a read to a global variable??? No clue why, but it is needed or else our accuracy drops to 50% which is the baseline accuracy which means we did obsolutely nothing. This problem also happens in `evict_reload2.c` but we are still able to accurately determine what the accessed address was just with less confidence (less of the trials registered hits then with the read to the global variable). Again no clue why this is the case. My only guess is maybe due to the timing of accessing a global variable (if so we should be able to reproduce similar behavior with a similar timed instruction)? Or its due to some micro arch memory changes that happen with a global variable access (but what is the different affects of global variable on memory)?
