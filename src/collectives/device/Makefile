#
# Copyright (c) 2015-2021, NVIDIA CORPORATION. All rights reserved.
#
# See LICENSE.txt for license information
#

include ../../../makefiles/common.mk
include ../../../makefiles/version.mk

BUILDDIR ?= $(abspath ../../../build)
OBJDIR := $(BUILDDIR)/obj/collectives/device

LIBSRCFILES := all_reduce.cu broadcast.cu reduce.cu all_gather.cu reduce_scatter.cu sendrecv.cu onerank_reduce.cu

LIBSRCFILES += functions.cu

DEPFILES   := $(patsubst %.cu, $(OBJDIR)/%.d, $(LIBSRCFILES))
DEPENDFILES:= $(DEPFILES:%.d=%.dep)
STATICLIB  := $(OBJDIR)/colldevice.a
DEVOBJ     := $(OBJDIR)/devlink.o
RULESFILE  := $(OBJDIR)/Makefile.rules

NVCUFLAGS  += -I. -I.. -I$(BUILDDIR)/include -I../../include
DEPFILEOUT := -o
ifneq '' '$(findstring clang,$(NVCC))'
  DEPFILEOUT := >
else
  NVCUFLAGS += --compiler-options "-fPIC -fvisibility=hidden"
endif


all: $(STATICLIB)

# Dummy rule so that the extra dependency (%.dep) files are preserved by make
all_deps: $(DEPENDFILES)

# Auto-generating the rules per op/reduction/datatype/algorithm
$(RULESFILE) : gen_rules.sh
	@printf "Generating %-35s > %s\n" rules $@
	@mkdir -p $(OBJDIR)
	@CUDA_MAJOR=${CUDA_MAJOR} CUDA_MINOR=${CUDA_MINOR} ./gen_rules.sh $(OBJDIR) > $@

-include $(RULESFILE)

LIBOBJ     := $(GENOBJS) $(OBJDIR)/functions.o $(OBJDIR)/onerank_reduce.o

-include $(DEPFILES)

$(STATICLIB): $(LIBOBJ) $(DEVOBJ)
	@printf "Archiving  %-35s > %s\n" objects $@
	ar cr $@ $^

# We do not want make to build *.d when running make clean.
# So we only provide targets for .dep which will produce .dep and .d,
# with only .d being included, and .dep keeping track of what needs to
# be regenerated.
$(OBJDIR)/%.dep : %.cu
	@mkdir -p $(OBJDIR)
	@$(NVCC) $(NVCUFLAGS) -M $< $(DEPFILEOUT) $@.tmp
	@sed "0,/^.*:/s//$(subst /,\/,$@):/" $@.tmp > $@
	@sed -e 's/.*://' -e 's/\\$$//' < $@.tmp | fmt -1 | \
                sed -e 's/^ *//' -e 's/$$/:/' >> $@
	@rm -f $@.tmp
	@cp $@ $(@:.dep=.d)

# Compiled kernels and collectives with relocatable device code ...
$(OBJDIR)/functions.o : functions.cu $(OBJDIR)/functions.dep
	@printf "Compiling  %-35s > %s\n" $< $@
	mkdir -p `dirname $@`
	$(NVCC) $(NVCUFLAGS) -fcuda-rdc -Xcuda-ptxas -maxrregcount=96 -c $< -o $@

$(OBJDIR)/onerank_reduce.o : onerank_reduce.cu $(OBJDIR)/onerank_reduce.dep
	@printf "Compiling  %-35s > %s\n" $< $@
	mkdir -p `dirname $@`
	$(NVCC) $(NVCUFLAGS) -fcuda-rdc -Xcuda-ptxas -maxrregcount=96 -c $< -o $@

# ... and create the device-side linked object with all those.
$(DEVOBJ) : $(LIBOBJ)
	nvlink --cpu-arch=X86_64 --arch=sm_60 --register-link-binaries=device_register_sm60.h --output-file=device_sm60.cubin $^
	fatbinary -64 --cmdline=-compile-only --link --compress-all --create=device.fatbin --embedded-fatbin=device_fatbin.h --image=profile=sm_60,file=device_sm60.cubin
	$(CXX) -c -x c++ -fPIC -DFATBINFILE="\"device_fatbin.h\"" -DREGISTERLINKBINARYFILE="\"device_register_sm60.h\"" -I. -D__NV_EXTRA_INITIALIZATION= -D__NV_EXTRA_FINALIZATION= -D__CUDA_INCLUDE_COMPILER_INTERNAL_HEADERS__ -pthread -fPIC "-I/usr/local/cuda/bin/../targets/x86_64-linux/include" -m64 "/usr/local/cuda/bin/crt/link.stub" -o $@

clean:
	rm -f $(LIBOBJ) $(DEVOBJ) $(DEPFILES) $(DEPENDFILES) $(RULESFILE) $(STATICLIB)
