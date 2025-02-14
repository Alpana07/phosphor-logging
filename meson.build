project('phosphor-logging', 'cpp',
    meson_version: '>= 0.57.0',
    default_options: [
      'buildtype=debugoptimized',
      'cpp_std=c++20',
      'warning_level=3',
      'werror=true',
      'libonly=' + (meson.is_subproject() ? 'true' : 'false'),
      'tests=' + (meson.is_subproject() ? 'disabled' : 'auto'),
    ],
    version: '1.0.0',
)

python_prog = find_program('python3', native: true)
systemd_dep = dependency('systemd')

sdbusplus_dep = dependency('sdbusplus', required: false)
sdbusplus_proj = dependency('', required: false)
if not sdbusplus_dep.found() or sdbusplus_dep.type_name() == 'internal'
    sdbusplus_proj = subproject('sdbusplus')
endif
if not sdbusplus_dep.found()
    sdbusplus_dep = sdbusplus_proj.get_variable('sdbusplus_dep')
endif

pdi_vars = []
if not get_option('openpower-pel-extension').disabled()
    pdi_vars += ['data_org_open_power=true']
endif
pdi_dep = dependency(
    'phosphor-dbus-interfaces',
    fallback: ['phosphor-dbus-interfaces', 'phosphor_dbus_interfaces_dep'],
    default_options: pdi_vars,
)

# Find the installed YAML directory, either from a configure option or
# by pulling it from the PDI dependency.
yamldir = get_option('yamldir')
if yamldir == ''
    yamldir = pdi_dep.get_variable(pkgconfig: 'yamldir', internal: 'yamldir')
endif

subdir('config')
subdir('tools')
subdir('lib')

if get_option('libonly')
  subdir_done()
endif

cpp = meson.get_compiler('cpp')

if sdbusplus_proj.found()
    sdbusplusplus_prog = sdbusplus_proj.get_variable('sdbusplusplus_prog')
    sdbuspp_gen_meson_prog = sdbusplus_proj.get_variable(
        'sdbuspp_gen_meson_prog')
else
    sdbusplusplus_prog = find_program('sdbus++', native: true)
    sdbuspp_gen_meson_prog = find_program('sdbus++-gen-meson', native: true)
endif

sdeventplus_dep = dependency(
    'sdeventplus',
    fallback: ['sdeventplus', 'sdeventplus_dep' ],
)

# Get Cereal dependency.
cereal_dep = dependency('cereal', required: false)
has_cereal = cpp.has_header_symbol(
    'cereal/cereal.hpp',
    'cereal::specialize',
    dependencies: cereal_dep,
    required: false)
if not has_cereal
    cereal_opts = import('cmake').subproject_options()
    cereal_opts.add_cmake_defines({'BUILD_TESTS': 'OFF'})
    cereal_proj = import('cmake').subproject(
        'cereal',
        options: cereal_opts,
        required: false)
    assert(cereal_proj.found(), 'cereal is required')
    cereal_dep = cereal_proj.dependency('cereal')
endif

# Generate sdbus++ files.
generated_sources = []
generated_others = []
subdir('gen')
subdir('gen/xyz')

# Generate callouts-gen.hpp.
callouts_gen = custom_target('callouts-gen.hpp'.underscorify(),
    input: [
        'callouts/callouts.py',
        'callouts/callouts-gen.mako.hpp',
        get_option('callout_yaml'),
    ],
    output: 'callouts-gen.hpp',
    command: [ python_prog, '@INPUT0@', '-i', '@INPUT2@', '-o', '@OUTPUT0@' ],
)
# Generate elog-lookup.cpp
elog_lookup_gen = custom_target('elog-lookup.cpp'.underscorify(),
    input: files(
        'tools/elog-gen.py',
        'tools/phosphor-logging/templates/elog-lookup-template.mako.cpp',
    ),
    output: 'elog-lookup.cpp',
    command: [
        python_prog, '@INPUT0@',
        '-t', '',
        '-m', '@INPUT1@',
        '-y', yamldir,
        '-u', meson.current_source_dir() / 'tools/',
        '-o', '@OUTPUT0@',
    ],
)
# Generate elog-process-metadata.cpp
elog_process_gen = custom_target('elog-process-metadata.cpp'.underscorify(),
    input: files(
        'tools/elog-gen.py',
        'tools/phosphor-logging/templates/elog-process-metadata.mako.cpp',
    ),
    output: 'elog-process-metadata.cpp',
    command: [
        python_prog, '@INPUT0@',
        '-t', '',
        '-m', '@INPUT1@',
        '-y', yamldir,
        '-u', meson.current_source_dir() / 'tools/',
        '-o', '@OUTPUT0@',
    ],
)

log_manager_ext_sources = []
log_manager_ext_deps = []
log_manager_ext_args = []

subdir('extensions')
subdir('phosphor-rsyslog-config')

# Generate daemon.
log_manager_sources = [
    generated_sources,
    callouts_gen,
    elog_lookup_gen,
    elog_process_gen,
    files(
        'elog_entry.cpp',
        'elog_meta.cpp',
        'elog_serialize.cpp',
        'extensions.cpp',
        'log_manager.cpp',
        'util.cpp',
    )
]
log_manager_deps = [
    cereal_dep,
    conf_h_dep,
    pdi_dep,
    phosphor_logging_dep,
    sdbusplus_dep,
    sdeventplus_dep,
]
executable('phosphor-log-manager',
    log_manager_sources,
    log_manager_ext_sources,
    'log_manager_main.cpp',
    include_directories: include_directories('gen'),
    cpp_args: log_manager_ext_args,
    dependencies: [
        log_manager_deps,
        log_manager_ext_deps,
    ],
    install: true,
)
# Generate test executables which run in obmc env (qemu, real hardware).
executable('logging-test',
    'logging_test.cpp',
    dependencies: [
        pdi_dep,
        phosphor_logging_dep,
        sdbusplus_dep,
    ],
    install: true,
)
executable('callout-test',
    'callouts/callout_test.cpp',
    dependencies: [
        conf_h_dep,
        pdi_dep,
        phosphor_logging_dep,
        sdbusplus_dep,
        sdeventplus_dep,
    ],
    install: true,
)

subdir('dist')

if not get_option('tests').disabled()
    subdir('test')
endif
