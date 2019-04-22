# Creating of custom board library

JIRA story: [MLTC-263](http://jira.msol.schneider-electric.com/browse/MLTC-263)

Relevant links and documents:
- [TI SDK RTOS: 3.1.3. Custom Board Addition](http://software-dl.ti.com/processor-sdk-rtos/esd/docs/latest/rtos/index_board.html#custom-board-addition)
- [AM65x_TRM_spruid7c.pdf](https://documents-svn.eur.gad.schneider-electric.com/svn/projects/02829_Multicarrier/03_DES_R+D/05_work/Datasheets/Sitara_AM65xx/AM65x_TRM_spruid7c.pdf)

***Note**
Tests were performed on TI-RTOS SDK 05.03(02) (PDK 1.0.4/3)

## Board Configurations ##

Board library supports different SoC and HW board specific configuration functions. For AM65xx Sitara SoC available are listed below:

* Pinmux configuration
* SoC Clock Settings
* DDR Configuration
* PLL Configuration
* Ethernet Configuration
* IO Instances
* Board Detection
* Board Flash APIs
* SerDes Configuration

Adding the custom board can be performed in two ways: by modifying existing board files or creating custom board in PDK build.
First approach represents quick hack and dirty approach so we will use the Second approach.

## Validating Board Configuration ##

Before updating the board library with configurations for custom board, it is recommended to use GEL file and CCS for validating the configurations.
The following should be set and confirmed:

* SoC clock configurations
* PLL clock configurations
* DDR PHY and timing configurations

## Creating Board Library with Custom Name ##

These steps are based on instructions in SDK documentation - [TI SDK RTOS: 3.1.3.6. Creating Board Library with Custom Name](http://software-dl.ti.com/processor-sdk-rtos/esd/docs/latest/rtos/index_board.html#creating-board-library-with-custom-name) 

However, instructions are based on AM572x platform so they are modified for AM65xx SoC. 
<br/>
#### 1. Creating new directories for custom board library ####
There are two directories in `<PDK_INSTALL_PATH>\packages\ti\board\src` that are associated with am65xx IDK board. They are `am65xx_idk` and `evmKeystone3`. Directory `evmKeystone3` contains common configuration for am65xx IDK and am65xx EVM boards like DDR settings, board ID, MMR etc. As we will probably change these settings on our board, files in `evmKeystone3` would also be modified. Thus the following must be performed:
* Copy `am65xx_idk` to `am65xx_mls` (or different suitable name)
* Copy `evmKeystone3` to `mlsKeystone3` (or different suitable name)
<br/>
#### 2. Updating names and makefile inside the customBoard package ####\

In `<PDK_INSTALL_PATH>\packages\ti\board\src\`, Rename files  and makefiles in copied directories to reflect new board name. Mekefile will need a bit of work depending on what elements of board you need for your platform.
In our case do the following:

* In directory `am65xx_mls` rename all src files from `*am65xx_idk*.*` to `*am65xx_mls*.*`
* In directory `am65xx_mls` in the makefile `src_files_am65xx_idk.mk` rename all source files and directories according to the changes above.
* In directory `mlsKeystone3` rename file `src_files_evmKeystone3.mk` to `src_files_mlsKeystone3.mk`
* In directory `mlsKeystone3` in the makefile `src_files_mlsKeystone3.mk` rename all soruce files and directories according to the changes above.
In directory `am65xx_mls` check include directives and macros and change `idk` to `mls`
* In `<PDK_INSTALL_PATH>\packages\ti\board\src\am65xx_mls\am65xx_mls_pinmux.c`	change to `#include "am65xx_mls_pinmux.h"`
* In `<PDK_INSTALL_PATH>\packages\ti\board\src\am65xx_mls\am65xx_mls_pinmux_data.c` change to `#include "am65xx_mls_pinmux.h"`
* In `<PDK_INSTALL_PATH>\packages\ti\board\src\am65xx_mls\include\board_cfg.h` change to `#define BOARD_INFO_BOARD_NAME 	"am65xx_mls"`

Also, to support NOR flash chips modify `<PDK_INSTALL_PATH>\packages\ti\board\src\flash\src_files_flash.mk` 
* Add `am65xx_mls` beside `am65xx_idk` 
<br/> 
#### 3. Adding MACRO based inclusion of updated board_cfg.h corresponding to custom Board ####  
In `<PDK_INSTALL_PATH>\packages\ti\board\board_cfg.h` , add the lines pointing to board_cfg.h file in your customBoard package. Thus, updated peripheral instances and board specific defines can be picked up.

- Add the following lines in `board_cfg.h`: 
```
#elif defined (am65xx_mls)
#include <ti\board\src\am65xx_mls\include\board_cfg.h>
```
<br/>
#### 4. Update top level board package makefile to include build for customBoard Library ####  
 The makefile is used to include all relevant make files for including Low level driver(LLD), source files relevant to board and the common board.c file. 
In `<PDK_INSTALL_PATH>\packages\ti\board\build\makefile.mk`, add board.c to the customBoard build:

- Modify by adding `am65xx_mls`
```
ifeq ($(BOARD),$(filter $(BOARD),evmAM335x icev2AM335x iceAMIC110 skAM335x bbbAM335x evmAM437x idkAM437x skAM437x evmAM572x idkAM571x idkAM572x evmK2H evmK2K evmK2E evmK2L evmK2G iceK2G evmC6678 evmC6657 evmOMAPL137 lcdkOMAPL138 idkAM574x am65xx_evm am65xx_idk am65xx_mls))
# Common source files across all platforms and cores
SRCS_COMMON += board.c
endif
```

- Add the following below `am65xx_evm am65xx_idk` part
```
ifeq ($(BOARD),$(filter $(BOARD), am65xx_mls))
include $(PDK_BOARD_COMP_PATH)/src/$(BOARD)/src_files_$(BOARD).mk
include $(PDK_BOARD_COMP_PATH)/src/mlsKeystone3/src_files_mlsKeystone3.mk
include $(PDK_BOARD_COMP_PATH)/src/flash/src_files_flash.mk
include $(PDK_BOARD_COMP_PATH)/src/src_files_lld.mk
endif
```
<br/>
#### 5. Update Global makerules ####  
File `build_config.mk` defines the global CFLAGS used to compile different PDK components. The SOC_AM65xx macro ensures that the CSL applicable to this SOC will be included in the build. Use the SoC name that corresponds to the platform of your custom board.  
In `<PDK_INSTALL_PATH>\packages\ti\build\makerules\build_config.mk` add line

- `CFLAGS_GLOBAL_am65xx_mls      = -DSOC_AM65XX -Dam65xx_mls=am65xx_mls`

**Optional step to update RTSC platform definition.** If you have a custom RTSC platform definition for your custom board that updates the memory and platform configuration using RTSC Tool then you need to update the platform.mk file that associates the RTSC platform with the corresponding board library

In `<PDK_INSTALL_PATH>\packages\ti\build\makerules\platform.mk`

* Add `am65xx_mls` beside `am65xx_idk`
<br/>
#### 6. Update Build makerules ####  

In `<PDK_INSTALL_PATH>\packages\ti\build\Rules.make` 

* Add `am65xx_mls` beside `am65xx_idk`
<br/>
#### 7. Update source files corresponding to drivers used in board library ####  

**This step is not needed for AM65xx but is left as a reference** File src_files_lld.mk adds source files corresponding to LLD drivers used in the board library. Usually most boards utilitize control driver like I2C (for programming the PMIC or reading EEPROM), UART drivers (for IO) and boot media drivers like (SPI/QSPI, MMC or NAND). In the example below, we assume that the custom Board library has dependency on I2C, SPI and UART LLD drivers. Since the LLD drivers will be linked to the application along with board library, board library only needs <driver>_soc.c corresponding to SOC used on the custom Board.

In `<PDK_INSTALL_PATH>\packages\ti\board\src\src_files_lld.mk`, add the following lines:

```
ifeq ($(BOARD),$(filter $(BOARD), myCustomBoard))
SRCDIR +=  $(PDK_INSTALL_PATH)/ti/drv/i2c/soc/am572x \
           $(PDK_INSTALL_PATH)/ti/drv/uart/soc/am572x \
           $(PDK_INSTALL_PATH)/ti/drv/spi/soc/am572x
		   
		   
INCDIR +=  $(PDK_INSTALL_PATH)/ti/drv/i2c/soc/am572x \
           $(PDK_INSTALL_PATH)/ti/drv/uart/soc/am572x \
           $(PDK_INSTALL_PATH)/ti/drv/spi/soc/am572x
```   
   
```		   
# Common source files across all platforms and cores
SRCS_COMMON += I2C_soc.c UART_soc.c SPI_soc.c
endif
```
<br/>
#### 8. Add custom Board to BOARDLIST and update CORELIST ####
In `<PDK_INSTALL_PATH>\packages\ti\board\board_component.mk`, modify the build to add your custom board and specify the cores for which you want to build the board library. Example to build board library for only A15 and C66x cores, limit the build by specify only a15_0 and C66x in the CORELIST

- Add am65xx_mls in the line
```
board_lib_BOARDLIST       = evmAM335x icev2AM335x iceAMIC110 skAM335x bbbAM335x evmAM437x idkAM437x skAM437x evmAM572x idkAM571x idkAM572x evmK2H evmK2K evmK2E evmK2L evmK2G iceK2G \
                            evmC6678 evmC6657 tda2xx-evm evmDRA75x tda2ex-evm evmDRA72x tda3xx-evm evmDRA78x evmOMAPL137 lcdkOMAPL138 idkAM574x am65xx_evm am65xx_idk am65xx_mls simJ7
```
<br/>
#### 9. Update .bld files for XDCTOOL based build steps ####
Make corresponding changes in `<PDK_INSTALL_PATH>\packages\ti\board\config.bld`, by adding the following lines:
```
var am65xx_mls = {
    name: "am65xx_mls",
    ccOpts: "-Dam65xx_mls -DSOC_AM65XX",
    targets: [A53LE],
}
```
```
/* List all the build targets here. */
Build.targets = [ C66LE, C66BE, A15LE, M4LE, A9LE, A8LE, ARM9LE, C674LE A53LE ];
var boards = [ evmAM335x, icev2AM335x, skAM335x, bbbAM335x, evmAM437x, idkAM437x, skAM437x, evmAM572x, idkAM571x, idkAM572x, evmK2H, evmK2K, evmK2E, evmK2L, evmK2G, evmC6678, evmC6657, evmOMAPL137 idkAM574x am65xx_evm am65xx_idk am65xx_mls];
```
Also, in `<PDK_INSTALL_PATH>\packages\ti\board\package.bld`, add the following line:
```
Pkg.otherFiles[Pkg.otherFiles.length++] = "src/mlsKeystone/src_files_mlsKeystone.mk";
```
<br/>   
#### 10. Setup Top level PDK build files to add the Custom board to setup environment ####
Modify  `<PDK_INSTALL_PATH>\packages\Rules.make` by adding 
```
export LIMIT_SOCS ?= am65xx
export LIMIT_BOARDS ?= am65xx_evm am65xx_idk am65xx_mls
```

## Bulding custom board library ##

To build package:
* change directory to `<PDK_INSTALL_PATH>\packages` 
* run `pdksetupenv.bat`
* To make just the board library: `gmake board_lib`

## Testing SBL build with custom board library ##

To build Secondary Bootloader with the custom board library SBL source and makefiles need to be modified.
* In `<PDK_INSTALL_PATH>\packages\ti\boot\sbl\sbl_component.mk` add `am65xx_mls`
* In `<PDK_INSTALL_PATH>\packages\ti\boot\sbl\src\mmcsd\sbl_mmcsd.c` add `am65xx_mls` in `FATFS_HwAttrs`
