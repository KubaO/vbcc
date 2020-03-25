cmake_minimum_required(VERSION 3.16)

project(vbcc C)
set(CMAKE_C_STANDARD 99)

set(TARGET i386 CACHE STRING "The target architecture of this compiler build")
#[=[
set_property(CACHE TARGET PROPERTY STRINGS
    alpha
    bi386
    c16x
    generic
    hc12
    i386
    m68k
    m68ks
    ppc
    qnice
    vidcore
    z
)
]=]

set(SRCDIR ${CMAKE_CURRENT_SOURCE_DIR})
set(TRGDIR ${SRCDIR}/machines/${TARGET})
set(TRGBINDIR ${CMAKE_BINARY_DIR}/${TARGET})

add_compile_definitions(
    #HAVE_AOS4
    #HAVE_ECPP
    #HAVE_MISRA
)

if (CMAKE_C_COMPILER_ID STREQUAL "GNU")
    add_compile_options(-fno-asm) # disables the asm keyword
endif()

#[=[
install: all
        $(INSTALL) -D bin/vc$X $(DESTDIR)/bin/vc$X
        $(INSTALL) -D bin/vprof$X $(DESTDIR)/bin/vprof$X
        $(INSTALL) -D bin/vbcc$(TARGET)$X $(DESTDIR)/bin/vbcc$(TARGET)$X

install-doc: doc/vbcc.pdf
        $(INSTALL) -D doc/vbcc.pdf $(DESTDIR)/share/doc/vbcc/vbcc.pdf

doc/vbcc.pdf:
        (cd doc; texi2dvi --pdf vbcc.texi; \
        rm -f vbcc.cp vbcc.fn vbcc.vr vbcc.tp vbcc.ky vbcc.pg vbcc.toc \
              vbcc.aux vbcc.log)

doc/vbcc.html:
        (cd doc;texi2html -split=chapter -nosec_nav -frames vbcc.texi)
        sed -e s/vbcc_13/vbcc_1/ <doc/vbcc_frame.html >doc/vbcc.html

vcppobjs = vcpp/cpp.o vcpp/eval.o vcpp/getopt.o vcpp/hideset.o vcpp/include.o \
           vcpp/lex.o vcpp/macro.o vcpp/nlist.o vcpp/tokens.o vcpp/unix.o

vbcc.tar.gz:
        (cd ..;tar zcvf vbcc.tar.gz vbcc/Makefile vbcc/*.[ch] vbcc/datatypes/*.[ch] vbcc/doc/*.texi vbcc/frontend/vc.c vbcc/machines/*/machine.[ch] vbcc/machines/*/machine.dt vbcc/machines/*/schedule.[ch] vbcc/ucpp/*.[ch] vbcc/ucpp/README vbcc/vprof/vprof.c vbcc/vsc/vsc.[ch])

dist: bin/osekrm$X
        mv supp.h t1
        mv supp.c t2
        mv main.c t3
        mv machines/ppc/machine.c t4
        mv declaration.c t5
        mv flow.c t6
        mv ic.c t7
        mv parse_expr.c t8
        mv statements.c t9
        mv rd.c t10
        mv type_expr.c t11
        bin/osekrm$X <t1 >supp.h
        bin/osekrm$X <t2 >supp.c
        bin/osekrm$X <t3 >main.c
        bin/osekrm$X <t4 >machines/ppc/machine.c
        bin/osekrm$X <t5 >declaration.c
        bin/osekrm$X <t6 >flow.c
        bin/osekrm$X <t7 >ic.c
        bin/osekrm$X <t8 >parse_expr.c
        bin/osekrm$X <t9 >statements.c
        bin/osekrm$X <t10 >rd.c
        bin/osekrm$X <t11 >type_expr.c
        make vbcc.tar.gz
        mv t1 supp.h
        mv t2 supp.c
        mv t3 main.c
        mv t4 machines/ppc/machine.c
        mv t5 declaration.c
        mv t6 flow.c
        mv t7 ic.c
        mv t8 parse_expr.c
        mv t9 statements.c
        mv t10 rd.c
        mv t11 type_expr.c
]=]

set(UCPPSOURCES
    ucpp/assert.c
    ucpp/cpp.c
    ucpp/eval.c
    ucpp/hash.c
    ucpp/lexer.c
    ucpp/macro.c
    ucpp/mem.c
)
set_property(SOURCE
    ${UCPPSOURCES}
    APPEND PROPERTY COMPILE_DEFINITIONS NO_UCPP_ERROR_FUNCTIONS
)
set_property(SOURCE
    ${UCPPSOURCES}
    APPEND PROPERTY INCLUDE_DIRECTORIES ${CMAKE_CURRENT_DIR}/ucpp
)

# Graph coloring register allocator by Alex
set(GCRASOURCES
    GCRegAlloc/Adjlist.c
    GCRegAlloc/DUChain.c
    GCRegAlloc/ICodeInsertion.c
    GCRegAlloc/NodeList.c
    GCRegAlloc/RAllocMain.c
    GCRegAlloc/Web.C
)
set_property(SOURCE
  ${GCRASOURCES}
  APPEND PROPERTY INCLUDE_DIRECTORIES GCRegAlloc ${TRGBINDIR}
)

set(BSOURCES
  declaration.c
  ic.c
  main.c
  parse_expr.c
  statements.c
  supp.c
  type_expr.c
  vars.c
  #{GCRASOURCES}
  ${UCPPSOURCES}
  ${TRGDIR}/machine.c
  ${TRGBINDIR}/dt.c
)

set(CSOURCES
  alias.c
  av.c
  cp.c
  cse.c
  flow.c
  loop.c
  opt.c
  rd.c
  regs.c
)

if (0)
set_property(SOURCE
  opt.c
  reg.c
  APPEND PROPERTY COMPILE_DEFINITIONS NO_OPTIMIZER
)
endif()

set(SSOURCES
  opt.c
  regs.c
  ${BSOURCES}
)

add_executable(vc frontend/vc.c)

add_executable(vprof vprof/vprof.c)

add_executable(osekrm osekrm.c)

add_executable(vbcc${TARGET}
  ${BSOURCES}
  ${CSOURCES}
)
target_include_directories(vbcc${TARGET} PRIVATE ${TRGDIR} ${TRGBINDIR} ${SRCDIR})

add_executable(vbccs${TARGET}
  ${SSOURCES}
  ${CSOURCES}
)
target_include_directories(vbccs${TARGET} PRIVATE ${TRGDIR} ${TRGBINDIR} ${SRCDIR})

if (EXISTS ${TRGDIR}/schedule.c)
    add_executable(vsc${TARGET}
        vsc/vsc.c
        ${TRGDIR}/schedule.c
    )
endif()

add_executable(dtgen
  datatypes/dtgen.c
)
target_include_directories(dtgen PRIVATE ${SRCDIR}/datatypes)

add_custom_command(
  OUTPUT ${TARGET}/dt.h ${TARGET}/dt.c
  COMMAND dtgen ${TRGDIR}/machine.dt ${TARGET}/dt.h ${TARGET}/dt.c < ${SRCDIR}/dtgen_msvc.in > NUL
  DEPENDS dtgen dtgen_msvc.in
)
set_property(SOURCE ${TRGBINDIR}/dt.c APPEND PROPERTY INCLUDE_DIRECTORIES ${TRGBINDIR} ${SRCDIR}/datatypes)

if (0)
add_executable(tasm${TARGET}
    tasm.c
    supp.c
    ${CSOURCES}
    ${TRGDIR}/machine.c
    ${TRGBINDIR}/dt.c
)
target_include_directories(tasm${TARGET} PRIVATE {SRCDIR}/vprof ${TRGDIR} ${TRGBINDIR} ${SRCDIR})
endif()

if (0)
add_executable(mbasic${TARGET}
    mbasic.c
    supp.c
    ${CSOURCES}
    ${TRGDIR}/machine.c
    ${TRGBINDIR}/dt.c
)
target_include_directories(mbasic${TARGET} PRIVATE ${TRGDIR} ${TRGBINDIR} ${SRCDIR})
endif()

if (0)
if (0)
    bison_target(parser minicompg.y minicompg.tab.c)
    # add ${BISON_parser_OUTPUTS} to the executable
endif()

add_executable(minicomp${TARGET}
    minicomp.c
    minicompg.tab.c
    supp.c
    ${CSOURCES}
    ${TRGDIR}/machine.c
    ${TRGBINDIR}/dt.c
)
target_include_directories(minicomp${TARGET} PRIVATE ${TRGBINDIR} ${TRGDIR})
endif()

if (0)
add_executable(ucpp
  ${UCPPSOURCES}
)
endif()


#[=[
$(TRGDIR)/minicompg.tab.o: minicompg.y minicomplexer.c minicomp.h supp.h
        bison minicompg.y
        $(CC) -c minicompg.tab.c -o $@ -I$(TRGDIR)
]=]
