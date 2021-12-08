# Backend-Project Synthesis-With-DFT

##8/12/2021 - by Sari##
##changed DFT settings##


##### synthesis script #####

puts -nonewline "\033\[1;31m"; #RED
puts "STRATING SYNTHESIS"
puts -nonewline "\033\[0m";# Reset
puts ""

set TopModule RStop

# List here ALL libraries you are going to use
# no need to add IO PADs libs here
# do not remove dw_foundation.sldb
# review link_library list with your supervisor

set link_library " dw_foundation.sldb \ 
/tools/kits/tower/PDK_TS18SL/FS120_STD_Cells_0_18um_2005_12/DW_TOWER_tsl18fs120/2005.12/synopsys/2004.12/models/tsl18fs120_typ.db /users/iit/tower_memories/rd3_768x8/rd3_768x8_typ.db "


# List here your target library ( libraries )
# target library is standard cell library which will be used to map your logic design (RTL)
# if you have more than one target library, make sure that all libraries are in the process corner ( typical or slow fast )
# review target_library list with your supervisor

set target_library \
"/tools/kits/tower/PDK_TS18SL/FS120_STD_Cells_0_18um_2005_12/DW_TOWER_tsl18fs120/2005.12/synopsys/2004.12/models/tsl18fs120_typ.db /users/iit/tower_memories/rd3_768x8/rd3_768x8_typ.db "



# List here your RTL ( VHDL / Verilog ) files to be synthesized
# defines and constants files should be listed first
# Top Module file should be just after defines and constants

analyze -library WORK -format sverilog {
../datain/RStop.v \
../datain/gf_inv.v \
../datain/gf_mult.v \
../datain/rd3_768x8.v \
../datain/syndromes_calc.v \
../datain/bm_engine.v \
../datain/rs_decoder.v \
../datain/chien_forney.v \
../datain/rs_encode.v }

elaborate ${TopModule} -architecture verilog -library WORK -update
current_design ${TopModule}

set filename "../report/post_elaborate.rpt"
redirect $filename { report_timing -delay_type max}
redirect -append $filename { report_timing -delay_type min}
redirect -append $filename { report_area }
redirect -append $filename { report_constraint -all_violators}
redirect -append $filename { check_design }

link

# read SDC

source ../scripts/${TopModule}.sdc
current_design ${TopModule}

# create input, output and feed through groups so the tool will focus on each path type separately
set ports_clock_root [filter_collection [get_attribute [get_clocks] sources] object_class==port]
group_path -name OUT -to [all_outputs]
group_path -name IN -from [remove_from_collection [all_inputs] $ports_clock_root]
group_path -name FEEDTHR -from [remove_from_collection [all_inputs] $ports_clock_root] -to [all_outputs]

# save initial DB for future debug

# write -format ddc -hierarchy -output ../dataout/initial.${TopModule}.ddc ${TopModule}

# Compile ultra settings
# leave unloaded sequential cells ( FFs )
# 1. good for early stages of design , when some FFs are mistakenly disconnected
# 2. good for last minute ECO when we want you use previously unused FFs

set compile_delete_unloaded_sequential_cells true
set compile_seqmap_propagate_constants false
set compile_seqmap_propagate_high_effort false

# enable constant propagation through combinatorial cells (not FFs ) -> realistic timing analysis

set case_analysis_with_logic_constants true
set template_separator_style "_"

# disable register merging , LEC will pass easier

set_register_merging [ get_designs ${TopModule} ] false  

# First Compile
puts -nonewline "\033\[1;31m"; #RED
puts "##### STRATING COMPILATION #####"
puts -nonewline "\033\[0m";# Reset
puts ""
compile
puts -nonewline "\033\[1;31m"; #RED
puts "##### FINISHED COMPILATION #####"
puts -nonewline "\033\[0m";# Reset
puts ""
# reports

 write -format ddc -hierarchy -output ../dataout/first.${TopModule}.ddc ${TopModule}

set filename "../report/post_compile.rpt"
redirect $filename { report_timing -delay_type max}
redirect -append $filename { report_timing -delay_type min}
redirect -append $filename { report_area }
redirect -append $filename { report_constraint -all_violators}
redirect -append $filename { check_design }

#enter scan chain, don't ignore Shift-registers

set_scan_configuration -style multiplexed_flip_flop
set compile_seqmap_identify_shift_registers false
#####
# create scan ports
create_port -dir in scan_en
create_port -dir in scan_reset 
create_port -dir in {scan_in1 scan_in2 scan_in3} 
create_port -dir out {scan_out1 scan_out2 scan_out3}
#create_clock  -period 10 -name clk -waveform {0 5} clk

set_dft_signal -view spec -type Reset -port scan_reset -active_state 1 
set_dft_signal -view existing_dft -type ScanClock -port OTN_SCLK_1_top -timing {45 55}; 
set_dft_signal -view existing_dft -type ScanEnable -port scan_en -active_state 1
set_dft_signal -view existing_dft -type ScanDataIn -port {scan_in1 scan_in2 scan_in3}
set_dft_signal -view existing_dft -type ScanDataOut -port {scan_out1 scan_out2 scan_out3}

set_scan_configuration -chain_count 3

#do scan synthesis
create_test_protocol -infer_asynch
dft_drc -verbose
set_dft_configuration -scan enable
set_dft_configuration -fix_set enable
insert_dft

#show all chains created in a file name scanfed
write_scan_def -output ../dataout/scandef

### write final files

#####write -format ddc -hierarchy -output ../dataout/final.${TopModule}.ddc ${TopModule}
set filename "../report/post_scan_chain.rpt"
redirect $filename { report_timing -delay_type max}
redirect -append $filename { report_timing -delay_type min}
redirect -append $filename { report_area }
redirect -append $filename { report_constraint -all_violators}
redirect -append $filename { check_design }

set verilogout_no_tri  true
set verilog_show_unconnected_pins  false
set verilog_unconnected_Prefix  true
set hdlout_internal_busses   true
set bus_inference_style  {%s[%d]}
set verilogout_single_bit  false
set bus_naming_style {%s[%d]}

write -format verilog -hierarchy -output ../dataout/RStop_post_synthesis.v ${TopModule}
write_sdc -nosplit ../dataout/${TopModule}.sdc

