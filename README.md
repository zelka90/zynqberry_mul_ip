# zynqberry_mul_ip
Плата: zynqberry (TE0726-03M)
Инструментальная машина с установленной ubuntu 16.04, Vivado Design Suite 2018.2, Petalinux 2018.2



Итого было проделанно

1. Создан проект в vivado содержащий процессорную систему и собственное ip ядро(п.2)
2.Создано ip ядро с интерфейсой AXIlite на 5 регистров на языке VHDL
2.1 Интерфейс 0 и 1 регистра без изменения, а в интерфейсе 2, 3 и 4 регистров убрана запись со стороны процессора:

	process (S_AXI_ACLK)
	variable loc_addr :std_logic_vector(OPT_MEM_ADDR_BITS downto 0); 
	begin
	  if rising_edge(S_AXI_ACLK) then 
	    if S_AXI_ARESETN = '0' then
	      slv_reg0 <= (others => '0');
	      slv_reg1 <= (others => '0');
--	      slv_reg2 <= (others => '0');
--	      slv_reg3 <= (others => '0');
--	      slv_reg4 <= (others => '0');
	    else
	      loc_addr := axi_awaddr(ADDR_LSB + OPT_MEM_ADDR_BITS downto ADDR_LSB);
	      if (slv_reg_wren = '1') then
	        case loc_addr is
	          when b"000" =>
	            for byte_index in 0 to (C_S_AXI_DATA_WIDTH/8-1) loop
	              if ( S_AXI_WSTRB(byte_index) = '1' ) then
	                -- Respective byte enables are asserted as per write strobes                   
	                -- slave registor 0
	                slv_reg0(byte_index*8+7 downto byte_index*8) <= S_AXI_WDATA(byte_index*8+7 downto byte_index*8);
	              end if;
	            end loop;
	          when b"001" =>
	            for byte_index in 0 to (C_S_AXI_DATA_WIDTH/8-1) loop
	              if ( S_AXI_WSTRB(byte_index) = '1' ) then
	                -- Respective byte enables are asserted as per write strobes                   
	                -- slave registor 1
	                slv_reg1(byte_index*8+7 downto byte_index*8) <= S_AXI_WDATA(byte_index*8+7 downto byte_index*8);
	              end if;
	            end loop;
--	          when b"010" =>
--	            for byte_index in 0 to (C_S_AXI_DATA_WIDTH/8-1) loop
--	              if ( S_AXI_WSTRB(byte_index) = '1' ) then
--	                -- Respective byte enables are asserted as per write strobes                   
--	                -- slave registor 2
--	                slv_reg2(byte_index*8+7 downto byte_index*8) <= S_AXI_WDATA(byte_index*8+7 downto byte_index*8);
--	              end if;
--	            end loop;
--	          when b"011" =>
--	            for byte_index in 0 to (C_S_AXI_DATA_WIDTH/8-1) loop
--	              if ( S_AXI_WSTRB(byte_index) = '1' ) then
--	                -- Respective byte enables are asserted as per write strobes                   
--	                -- slave registor 3
--	                slv_reg3(byte_index*8+7 downto byte_index*8) <= S_AXI_WDATA(byte_index*8+7 downto byte_index*8);
--	              end if;
--	            end loop;
--	          when b"100" =>
--	            for byte_index in 0 to (C_S_AXI_DATA_WIDTH/8-1) loop
--	              if ( S_AXI_WSTRB(byte_index) = '1' ) then
--	                -- Respective byte enables are asserted as per write strobes                   
--	                -- slave registor 4
--	                slv_reg4(byte_index*8+7 downto byte_index*8) <= S_AXI_WDATA(byte_index*8+7 downto byte_index*8);
--	              end if;
--	            end loop;
	          when others =>
	            slv_reg0 <= slv_reg0;
	            slv_reg1 <= slv_reg1;
--	            slv_reg2 <= slv_reg2;
--	            slv_reg3 <= slv_reg3;
--	            slv_reg4 <= slv_reg4;
	        end case;
	      end if;
	    end if;
	  end if;                   
	end process; 

2.2 объявлены внутренние сигнал:
	---- Signals for user logic register space example
	signal in1_mul : std_logic_vector(31 downto 0);
	signal in2_mul : std_logic_vector(31 downto 0);
	signal out_mul : std_logic_vector(63 downto 0);
	--------------------------------------------------
2.3 Реализована логика умножителя и подключена к регистрам AXI
	-- Add user logic here
	    in1_mul <= slv_reg0;
	    in2_mul <= slv_reg1;
	    out_mul <= std_logic_vector(unsigned(in1_mul)*unsigned(in2_mul));
	    slv_reg2 <= out_mul(31 downto 0);
	    slv_reg3 <= out_mul(63 downto 32);
	    slv_reg4 <= x"00000011";
	-- User logic ends
примечание: в регистр 4 записана константа для отслеживания обращения к нужной области памяти.

3. Проект собран и экспортирован в SDK, проверена его работа

4. Собран образ системы и скопирован на флешку:
petalinux-create --type project --template zynq --name OS_myIpv3

cd OS_myIpv3/

petalinux-config --get-hw-description=/home/anzhelika/projekt/myIpv3/VivadoDesign/VivadoDesign.sdk/Diagramm_wrapper_hw_platform_0/

--------------------------------------------------------------
конфигурация системы

Subsystem AUTO Hardware Settings
    SD/SDIO Settings
        Change Primary SD/SDIO to ps7_sd_1. 
    Advanced bootable images storage Settings
        dtb image settings
            Set Image Storage Media to primary sd.  
Image Packaging Configuration
    Change Root filesystem type to SD card
    Verify Device node of SD device is set to /dev/mmcblk0p2
    Uncheck Copy final images to tftpboot
    
---------------------------------------------------------------

petalinux-config -c kernel

In the kernel config menus
    Device drivers
        Network device support
            USB Network adapters
                Include Multi-purpose USB Networking Framework
                    Include SMSC LAN95XX based USB2.0 10/100 ethernet devices  

nano ./project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi
добавить следующие строки в файл:
/{
    usb_phy0: usb_phy@0 {
        compatible = "ulpi-phy";
        #phy-cells = <0>;
        reg = <0xe0002000 0x1000>;
        view-port = <0x0170>;
        drv-vbus;
    };
};
 
&usb0 {
    usb-phy = <&usb_phy0>;
} ;

#собрать образ
petalinux-build

#загрузочный образ пакета собержащий загрузчик первого уровня fsbl, fpga битовый поток и U-BOOT

# генерация BOOT.bin файла

petalinux-package --boot --fsbl ./images/linux/zynq_fsbl.elf --fpga /home/anzhelika/projekt/myIpv3/VivadoDesign/VivadoDesign.sdk/Diagramm_wrapper_hw_platform_0/Diagramm_wrapper.bit  --u-boot

#упаковать все в prebuild директорию для удобства
petalinux-package --prebuilt --fpga /home/anzhelika/projekt/myIpv3/VivadoDesign/VivadoDesign.sdk/Diagramm_wrapper_hw_platform_0/Diagramm_wrapper.bit 

#Подготавливаем флешку
#запускаем программу
sudo gparted
#разбиваем флешку на 2 раздела 1- fat32 min 48MB (сделала 512) поставить метку lba
#2 - оставшееся место ex4

#создать директории
sudo mkdir /media/boot
sudo mkdir /media/rootfs
 
#Примонтировать разделы флешки
sudo mount /dev/mmcblk0p1 /media/boot
sudo mount /dev/mmcblk0p2 /media/rootfs

#копируем файлы на флешку в fat раздел
sudo cp pre-built/linux/images/BOOT.BIN  /media/boot
sudo cp pre-built/linux/images/system.dtb  /media/boot
sudo cp images/linux/image.ub  /media/boot

#копирование файловой системы ubuntu на карту
sudo tar xfvp ~/Загрузки/ubuntu-16.04.2-minimal-armhf-2017-06-18/armhf-rootfs-ubuntu-xenial.tar -C /media/rootfs/

sync
sudo chown root:root /media/rootfs/
sudo chmod 755 /media/rootfs/ 

#Add file systems table
sudo sh -c "echo '/dev/mmcblk0p2  /  auto  errors=remount-ro  0  1' >> /media/rootfs/etc/fstab"
sudo sh -c "echo '/dev/mmcblk0p1  /boot/uboot  auto  defaults  0  2' >> /media/rootfs/etc/fstab"

#Unmount SD card
sudo umount /media/boot
sudo umount /media/rootfs

5. Делать быстро: Вставить флешку в плату, подключить к компьютеру, включить терминал com порта, в моем случае minicom.
Пока на экране выводятся цыфры от 5 до 0 нажать enter и ввести следующее: 

setenv cp_dtb2ram 'fatload mmc 1 ${dtbnetstart} ${dtb_img}'                                                            
setenv cp_kernel2ram 'fatload mmc 1 ${netstart} ${kernel_img}'                                                          
setenv default_bootcmd 'run cp_kernel2ram && run cp_dtb2ram && bootm ${netstart} - ${dtbnetstart}'
setenv bootargs 'console=ttyPS0,115200 earlyprintk root=/dev/mmcblk0p2 rw rootwait'
saveenv
boot

6. После загрузки операционной системы вводится имя пользователя ubuntu и пароль temppwd(есть подсказки на экране)
7. обновялем список пакетов:
	sudo apt-get update
8. устанавливаем новые пакеты
	sudo apt-get upgrade
9. устанавливаем пакет:
	sudo apt-get install build-essential
10. переходим в пользовательскую папку и создаем файл test.c со следующим содержанием:
#include <sys/mman.h>
#include <stdio.h>
#include <stdint.h>
#include <unistd.h>
#include <stdlib.h>
#include <getopt.h>
#include <fcntl.h>
#include <sys/ioctl.h>
#include <linux/types.h>
#include <linux/spi/spidev.h>
#include <errno.h>

uint32_t BaseAddres=0x43c00000;
uint32_t mem_size=0x43c00004;

int sendRegData(uint32_t mem_address, int value){
    int mem_dev = open("/dev/mem", O_RDWR | O_SYNC);

    if(mem_dev == -1){
     printf("Failed to open /dev/mem sendRegData\n");
       return -1;
    }

    uint32_t alloc_mem_size, page_mask, page_size;
    void *mem_pointer, *virt_addr;

    page_size = sysconf(_SC_PAGESIZE);
    alloc_mem_size = (((mem_size / page_size) + 1) * page_size);
      page_mask = (page_size - 1);

    mem_pointer = mmap(NULL,
                       alloc_mem_size,
                       PROT_READ | PROT_WRITE,
                       MAP_SHARED,
                       mem_dev,
                       (mem_address & ~page_mask)
                       );

      if(mem_pointer == MAP_FAILED){
          return -1;
    }                       

    virt_addr = (mem_pointer + (mem_address & page_mask));
    volatile unsigned int* p = (volatile unsigned int*)virt_addr;
    p[0] = value;              

    munmap(mem_pointer, alloc_mem_size);
    close(mem_dev);
     return value;
}

int readRegData(uint32_t mem_address) {
    int mem_dev = open("/dev/mem", O_RDWR | O_SYNC);

    if(mem_dev == -1) {
       printf("Failed to open /dev/mem readRegData\n");
       return -1;
    }

     uint32_t alloc_mem_size, page_mask, page_size;
    void *mem_pointer, *virt_addr;

    page_size = sysconf(_SC_PAGESIZE);
    alloc_mem_size = (((mem_size / page_size) + 1) * page_size);
    page_mask = (page_size - 1);

    mem_pointer = mmap(NULL,
                       alloc_mem_size,
                        PROT_READ | PROT_WRITE,
                       MAP_SHARED,
                       mem_dev,
                       (mem_address & ~page_mask)
                       );

    if(mem_pointer == MAP_FAILED) {
          printf("Failed to map memory\n");
          return -1;
    }
      virt_addr = (mem_pointer + (mem_address & page_mask));
    volatile unsigned int* p = (volatile unsigned int*)virt_addr;
        int rtrn = p[0];    
   int munmprtrn = munmap(mem_pointer, alloc_mem_size);   
close(mem_dev);

    return rtrn;
  }

   int main(int argc, void *argv[]){
	int readReg[5];	
	if (argc != 3)	{
		printf("invalid number of arguments\n");
		return 0;
	}
	int in1 =atoll(argv[1]);
	int in2 =atoll(argv[2]);
	long out;
	sendRegData(BaseAddres,in1);
        sendRegData(BaseAddres+4,in2);
        for(int i=0;i<5;i++){
                readReg[i]=readRegData(BaseAddres+i*4);
                printf("reg%d=%d\n",i,readReg[i]);
        }
	out=((long)readReg[3] << 31) | (long)readReg[2];
	printf("%d * %d = %lu\n",readReg[0],readReg[1],(unsigned long)out);

}

11. Собираем файл
	gcc test.c
12. В SDK запускаем programm FPGA
13.Запускаем приложение от имени администратора
	sudo ./a.out 2 3
и видим результат работы:                                         
	reg0=2                                                                          
	reg1=3                                                                          
	reg2=6                                                                          
	reg3=0                                                                          
	reg4=17                                                                         
	2 * 3 = 6 
