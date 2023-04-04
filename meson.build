project('radius-auth', 'c', 'cpp',
    version : '1.0',
    meson_version: '>=0.57.0',
    default_options : [
        'buildtype=debugoptimized',
        'c_std=gnu11',
        'cpp_std=c++20'
    ]
)

# Include directories
incdir = include_directories(
    'service_files',
    'include',
)

# Include dependent library
crashdump_dependencies = []
cjson = meson.get_compiler('cjson')


# Include source file
crashdump_src = files(
    'utils_triage.cpp',
    'utils_dbusplus.cpp',
    'crashdump.cpp',
    'engine/base64Encode.c',
    'engine/BigCore.c',
    'engine/cmdprocessor.c',
    'engine/crashdump.c',
    'engine/Crashlog.c',
    'engine/flow.c',
    'engine/inputparser.c',
    'engine/logger.c',
    'engine/nvd.c',
    'engine/TorDump.c',
    'engine/utils.c',
    'engine/validator.c'
)

# Make dynamic library
#shared_library( 
    #'openapml', 
    #apml_src ,
    #dependencies : apml_dependencies,
    #include_directories : incdir,
    #name_prefix : '',
    #install : true,
    #install_dir : '/lib/security'
#)

executable(
    'crashdump',
    crashdump_src,
    include_directories : incdir,
    install_dir : '/usr/bin/'
)