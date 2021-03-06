# This file is part of libsigc++.

project('libsigc++', 'cpp',
  version: '3.0.1',
  license: 'LGPLv2.1+',
  default_options: [
    'cpp_std=c++17',
    'warning_level=0',
  ],
  meson_version: '>= 0.50.0', # required for python3.path()
)

sigcxx_api_version = '3.0'
sigcxx_pcname = 'sigc++-' + sigcxx_api_version

sigcxx_version_array = meson.project_version().split('.')
sigcxx_major_version = sigcxx_version_array[0].to_int()
sigcxx_minor_version = sigcxx_version_array[1].to_int()
sigcxx_micro_version = sigcxx_version_array[2].to_int()

# http://www.gnu.org/software/libtool/manual/html_node/Updating-version-info.html
# The relation between libtool's current:revison:age interface versioning
# and the .so filename, .so.x.y.z, is
# x = current - age
# y = age
# z = revision
# If libtool_soversion is updated as described in libtool's documentation,
# x.y.z will usually *not* be equal to meson.project_version().
libtool_soversion = [0, 0, 0]
sigcxx_libversion = '@0@.@1@.@2@'.format(
  libtool_soversion[0] - libtool_soversion[2],
  libtool_soversion[2],
  libtool_soversion[1])

# Use these instead of meson.source_root() and meson.build_root() in subdirectories.
# source_root() and build_root() are not useful, if this is a subproject.
project_source_root = meson.current_source_dir()
project_build_root = meson.current_build_dir()

cpp_compiler = meson.get_compiler('cpp')
is_msvc = cpp_compiler.get_id() == 'msvc'
python3 = import('python').find_installation()

python_version = python3.language_version()
python_version_req = '>= 3.5'
if not python_version.version_compare(python_version_req)
  error('Requires Python @0@, found @1@.'.format(python_version_req, python_version))
endif

# Do we build from a git repository?
# Suppose we do if and only if a '.git' directory or file exists.
cmd_py = '''
import os
import sys
sys.exit(os.path.isdir("@0@") or os.path.isfile("@0@"))
'''.format(project_source_root / '.git')
is_git_build = run_command(python3, '-c', cmd_py).returncode() != 0

# Options.
maintainer_mode_opt = get_option('maintainer-mode')
maintainer_mode = maintainer_mode_opt == 'true' or \
                 (maintainer_mode_opt == 'if-git-build' and is_git_build)
warning_level = get_option('warnings')
build_deprecated_api = get_option('build-deprecated-api')
build_documentation_opt = get_option('build-documentation')
build_documentation = build_documentation_opt == 'true' or \
                     (build_documentation_opt == 'if-maintainer-mode' and maintainer_mode)
build_examples = get_option('build-examples')
do_benchmark = get_option('benchmark')

# Installation directories are relative to {prefix}.
install_prefix = get_option('prefix')
install_includedir = get_option('includedir')
install_libdir = get_option('libdir')
install_datadir = get_option('datadir')
install_pkgconfigdir = install_libdir / 'pkgconfig'

# Dependencies.
# sigcxx_build_dep: Dependencies when building the libsigc++ library.
# sigcxx_dep (created in sigc++/meson.build):
#   Dependencies when using the libsigc++ library.
sigcxx_build_dep = [] # No dependencies

benchmark_dep = dependency('boost', modules: ['system', 'timer'],
                           version: '>=1.20.0', required: do_benchmark)
can_benchmark = benchmark_dep.found()

if is_msvc
  # We must have Visual Studio 2017 15.7 or later...
  assert(cpp_compiler.version().split('.')[0].to_int() >= 19 and \
         cpp_compiler.version().split('.')[1].to_int() >= 15,
         'Visual Studio 2017 15.7 or later is required')
endif

# Some dependencies are required only in maintainer mode and/or
# if documentation shall be built.
mm_common_get = find_program('mm-common-get', required: false)

if maintainer_mode and not mm_common_get.found()
  error('Maintainer mode requires the \'mm-common-get\' command.\n' +
        'Use \'-Dmaintainer-mode=false\' or install the \'mm-common\' package, version 1.0.0 or higher')
endif

perl = find_program('perl', required: build_documentation)
doxygen = find_program('doxygen', required: build_documentation)
dot = find_program('dot', required: build_documentation) # Used by Doxygen
xsltproc = find_program('xsltproc', required: build_documentation)

script_dir = project_source_root / 'untracked' / 'build_scripts'
doc_reference = script_dir / 'doc-reference.py'
dist_changelog = script_dir / 'dist-changelog.py'
dist_build_scripts = script_dir / 'dist-build-scripts.py'
dist_cmd = project_source_root / 'tools' / 'dist-cmd.py' # Must be committed to git.
tutorial_custom_cmd = project_source_root / 'tools' / 'tutorial-custom-cmd.py'

if maintainer_mode and mm_common_get.found()
  # Copy files to untracked/build_scripts and untracked/docs/docs.
  run_command(mm_common_get, '--force', script_dir,
    project_source_root / 'untracked' / 'docs' / 'docs')
endif

# Set compiler warnings.
warning_flags = []
if warning_level == 'min'
  if is_msvc
    warning_flags = ['/W3']
  else
    warning_flags = ['-Wall']
  endif
elif warning_level == 'max' or warning_level == 'fatal'
  if is_msvc
    warning_flags = ['/W4']
  else
    warning_flags = '-pedantic -Wall -Wextra -Wsuggest-override -Wshadow -Wzero-as-null-pointer-constant -Wformat-security'.split()
  endif
  if warning_level == 'fatal'
    if is_msvc
      warning_flags += ['/WX']
    else
      warning_flags += ['-Werror']
    endif
  endif
endif

warning_flags = cpp_compiler.get_supported_arguments(warning_flags)
add_project_arguments(warning_flags, language: 'cpp')

# MSVC: Ignore warnings that aren't really harmful, but make those
#       that should not be overlooked stand out.
if is_msvc
  foreach wd : ['/FImsvc_recommended_pragmas.h', '/wd4267', '/wd4530', '/utf-8']
    disabled_warning = cpp_compiler.get_supported_arguments(wd)
    add_project_arguments(disabled_warning, language: 'cpp')
  endforeach
endif

# Configure files
pkg_conf_data = configuration_data()
pkg_conf_data.set('prefix', install_prefix)
pkg_conf_data.set('exec_prefix', '${prefix}')
pkg_conf_data.set('libdir', '${exec_prefix}' / install_libdir)
pkg_conf_data.set('datarootdir', '${prefix}' / install_datadir)
pkg_conf_data.set('datadir', '${datarootdir}')
pkg_conf_data.set('includedir', '${prefix}' / install_includedir)
pkg_conf_data.set('top_srcdir', project_source_root)
pkg_conf_data.set('PACKAGE_VERSION', meson.project_version())
pkg_conf_data.set('SIGCXX_API_VERSION', sigcxx_api_version)

if not build_deprecated_api
  pkg_conf_data.set('SIGCXX_DISABLE_DEPRECATED', true)
endif
pkg_conf_data.set('SIGCXX_MAJOR_VERSION', sigcxx_major_version)
pkg_conf_data.set('SIGCXX_MINOR_VERSION', sigcxx_minor_version)
pkg_conf_data.set('SIGCXX_MICRO_VERSION', sigcxx_micro_version)

configure_file(
  input: 'sigc++.pc.in',
  output: sigcxx_pcname + '.pc',
  configuration: pkg_conf_data,
  install_dir: install_pkgconfigdir,
)

configure_file(
  input: 'sigc++-uninstalled.pc.in',
  output: sigcxx_pcname + '-uninstalled.pc',
  configuration: pkg_conf_data,
)

install_includeconfigdir = install_libdir / sigcxx_pcname / 'include'
configure_file(
  input: 'sigc++config.h.meson',
  output: 'sigc++config.h',
  configuration: pkg_conf_data,
  install_dir: install_includeconfigdir,
)

#subdir('cmake')
subdir('MSVC_NMake')
subdir('sigc++')
subdir('examples')
subdir('tests')
subdir('docs/docs/reference')
subdir('docs/docs/manual')

if not meson.is_subproject()
  # Add a ChangeLog file to the distribution directory.
  # (add_dist_script() is not allowed in a subproject)
  meson.add_dist_script(
    python3.path(), dist_cmd,
    python3.path(), dist_changelog,
    project_source_root,
  )
  # Add build scripts to the distribution directory, and delete .gitignore
  # files and an empty $MESON_DIST_ROOT/build/ directory.
  meson.add_dist_script(
    python3.path(), dist_cmd,
    python3.path(), dist_build_scripts,
    project_source_root,
    'untracked' / 'build_scripts',
  )
endif

# Print a summary.
real_maintainer_mode = ''
if maintainer_mode_opt == 'if-git-build'
  real_maintainer_mode = ' (@0@)'.format(maintainer_mode)
endif

real_build_documentation = ''
if build_documentation_opt == 'if-maintainer-mode'
  real_build_documentation = ' (@0@)'.format(build_documentation)
endif

validate = get_option('validation') and can_parse_and_validate
explain_val = ''
if get_option('validation') and not validate
  explain_val = ' (requires xmllint)'
endif

build_pdf = build_pdf_by_default and can_build_pdf
explain_pdf = ''
if build_pdf_by_default and not build_pdf
  explain_pdf = ' (requires dblatex or (xmllint and docbook2pdf))'
endif



summary = [
  '',
  '------',
  meson.project_name() + ' ' + meson.project_version(),
  '',
  '         Maintainer mode: @0@@1@'.format(maintainer_mode_opt, real_maintainer_mode),
  '       Compiler warnings: @0@'.format(warning_level),
  '    Build deprecated API: @0@'.format(build_deprecated_api),
  'Build HTML documentation: @0@@1@'.format(build_documentation_opt, real_build_documentation),
  '          XML validation: @0@@1@'.format(validate, explain_val),
  '               Build PDF: @0@@1@'.format(build_pdf, explain_pdf),
  '  Build example programs: @0@'.format(build_examples),
  '               Benchmark: @0@'.format(do_benchmark),
  'Directories:',
  '                  prefix: @0@'.format(install_prefix),
  '              includedir: @0@'.format(install_prefix / install_includedir),
  '        includesigcxxdir: @0@'.format(install_prefix / install_includedir / sigcxx_pcname),
  '                  libdir: @0@'.format(install_prefix / install_libdir),
  '        includeconfigdir: @0@'.format(install_prefix / install_includeconfigdir),
  '            pkgconfigdir: @0@'.format(install_prefix / install_pkgconfigdir),
  '                 datadir: @0@'.format(install_prefix / install_datadir),
  '                  docdir: @0@'.format(install_prefix / install_docdir),
  '              devhelpdir: @0@'.format(install_prefix / install_devhelpdir),
  '             tutorialdir: @0@'.format(install_prefix / install_tutorialdir),
  '------'
]

message('\n'.join(summary))
