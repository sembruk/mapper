#
#    Copyright 2012, 2013 Thomas Schöps
#    Copyright 2012-2017 Kai Pastor
#    
#    This file is part of OpenOrienteering.
# 
#    OpenOrienteering is free software: you can redistribute it and/or modify
#    it under the terms of the GNU General Public License as published by
#    the Free Software Foundation, either version 3 of the License, or
#    (at your option) any later version.
# 
#    OpenOrienteering is distributed in the hope that it will be useful,
#    but WITHOUT ANY WARRANTY; without even the implied warranty of
#    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#    GNU General Public License for more details.
# 
#    You should have received a copy of the GNU General Public License
#    along with OpenOrienteering.  If not, see <http://www.gnu.org/licenses/>.
 
message(STATUS "Configuring ${PROJECT_NAME} translations")

find_package(Qt5LinguistTools REQUIRED)

# Translation files which are updated by lupdate.
set(Mapper_translations
  OpenOrienteering_cs.ts
  OpenOrienteering_da.ts
  OpenOrienteering_de.ts
  OpenOrienteering_en.ts
  OpenOrienteering_eo.ts
  OpenOrienteering_es.ts
  OpenOrienteering_fi.ts
  OpenOrienteering_fr.ts
  OpenOrienteering_hu.ts
  OpenOrienteering_id.ts
  OpenOrienteering_it.ts
  OpenOrienteering_ja.ts
  OpenOrienteering_lv.ts
  OpenOrienteering_nb.ts
  OpenOrienteering_nl.ts
  OpenOrienteering_pl.ts
  OpenOrienteering_pt_BR.ts
  OpenOrienteering_ru.ts
  OpenOrienteering_sv.ts
  OpenOrienteering_uk.ts
  OpenOrienteering_zh_CN.ts
)

# Translation files which are updated but not deployed.
set(Mapper_translations_template
  OpenOrienteering_et.ts
  OpenOrienteering_eu.ts
  OpenOrienteering_he.ts
  OpenOrienteering_template.ts
)

# Translation files which are updated by SymbolSetTool.
set(map_symbols_translations
  map_symbols_cs.ts
  map_symbols_da.ts
  map_symbols_de.ts
  map_symbols_eo.ts
  map_symbols_fi.ts
  map_symbols_fr.ts
  map_symbols_nl.ts
  map_symbols_ru.ts
  map_symbols_sv.ts
  map_symbols_uk.ts
)

# Translation files which are updated but not deployed.
set(map_symbols_translations_template
  map_symbols_template.ts
)

# Translation files which are maintained by hand.
set(qt_translations
  qt_eo.ts
  qt_et.ts
  qt_id.ts
  qt_lv.ts
  qt_nb.ts
  qt_nl.ts
)

qt5_add_translation(translations_qm ${Mapper_translations} ${map_symbols_translations} ${qt_translations})

# A target which builds all translations.
add_custom_target(Mapper_translations ALL DEPENDS ${translations_qm})

install(
  FILES ${translations_qm}
  DESTINATION "${MAPPER_DATA_DESTINATION}/translations"
)

if(APPLE)
	# Localized resources just to enable localized native dialogs.
	foreach(mapper_trans en ${Mapper_translations})
		get_filename_component(mapper_lang ${mapper_trans} NAME_WE)
		string(REPLACE OpenOrienteering_ "" language_code ${mapper_lang})
		configure_file(locversion.plist.in ${language_code}.lproj/locversion.plist @ONLY)
		install(
		  DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/${language_code}.lproj"
		  DESTINATION "${MAPPER_DATA_DESTINATION}")
	endforeach()
endif()

# For super-builds (e.g. cross-compilation), wait for the Qt build.
add_dependencies(Mapper_translations Mapper_prerequisites)



# We need lupdate >= 5.6.1 for updates from source
if(TARGET ${Qt5_LUPDATE_EXECUTABLE})
	get_property(Qt5_LUPDATE_EXECUTABLE TARGET ${Qt5_LUPDATE_EXECUTABLE} PROPERTY IMPORTED_LOCATION)
endif()
execute_process(
  COMMAND "${Qt5_LUPDATE_EXECUTABLE}" -version
  OUTPUT_VARIABLE lupdate_version
)
string(REGEX REPLACE "^.*([0-9]\\.[0-9]\\.[0-9]).*" "\\1" lupdate_version "${lupdate_version}")
if(lupdate_version VERSION_LESS 5.6.1)
	message(AUTHOR_WARNING "lupdate version ${lupdate_version} is less than 5.6.1. Updates disabled.")
else()
	# A target which updates all translations.
	add_custom_target(Mapper_translations_update)
	
	set(listfile "${CMAKE_CURRENT_BINARY_DIR}/translations_sourcefiles.txt")
	set_property(SOURCE "${listfile}" PROPERTY GENERATED TRUE)
	set_property(TARGET Mapper_translations_update PROPERTY listfile "${listfile}")
	file(WRITE "${listfile}"
	  "${PROJECT_SOURCE_DIR}/translations/future_translations.cpp\n"
	  "${PROJECT_SOURCE_DIR}/packaging/translations.cpp\n"
	)
	
	# One target for each translation.
	foreach(translation ${Mapper_translations} ${Mapper_translations_template})
		set(lupdate_options -locations absolute)
		get_filename_component(ts_filename ${translation} NAME_WE)
		if("${ts_filename}" STREQUAL "OpenOrienteering_en")
			list(APPEND lupdate_options -pluralonly)
		endif()
		if("${ts_filename}" STREQUAL "OpenOrienteering_template")
			list(APPEND lupdate_options -no-obsolete)
		endif()
		add_custom_target(Mapper_${ts_filename}_update
		  # SOURCES are displayed in Qt Creator
		  SOURCES "${PROJECT_SOURCE_DIR}/translations/future_translations.cpp"
		          "${PROJECT_SOURCE_DIR}/packaging/translations.cpp"
		          "${listfile}"
		  COMMAND "${Qt5_LUPDATE_EXECUTABLE}" ${lupdate_options} "@${listfile}" -ts "${CMAKE_CURRENT_SOURCE_DIR}/${translation}"
		  COMMAND sed -i -e "/<name>Q[A-Z]/,/<context>/d" "${CMAKE_CURRENT_SOURCE_DIR}/${translation}"
		  VERBATIM
		)
		add_dependencies(Mapper_translations_update Mapper_${ts_filename}_update)
	endforeach()
endif()

#
# A macro for collecting translations sources.
#
# Synopsis:
#
# MAPPER_TRANSLATIONS_SOURCES(a.cpp b.cpp)
#
macro(MAPPER_TRANSLATIONS_SOURCES)
	if (TARGET Mapper_translations_update)
		get_property(listfile TARGET Mapper_translations_update PROPERTY listfile)
		foreach(arg ${ARGN})
			get_source_file_property(_abs_path ${arg} LOCATION)
			file(APPEND "${listfile}" "${_abs_path}\n")
		endforeach()
	endif()
endmacro()



if(UNIX AND NOT APPLE)
	set(update_source )
	if(Mapper_DEVELOPMENT_BUILD)
		# Only development builds are allowed to update the source directories.
		set(update_sources
		  COMMAND "${CMAKE_COMMAND}" -E copy_if_different "../packaging/linux/Mapper.desktop" "${PROJECT_SOURCE_DIR}/packaging/linux/"
		  COMMAND "${CMAKE_COMMAND}" -E copy_if_different "../packaging/linux/openorienteering-mapper.xml" "${PROJECT_SOURCE_DIR}/packaging/linux/"
		)
	endif()
	set(translations )
	foreach(translation ${Mapper_translations})
		list(APPEND translations "${CMAKE_CURRENT_SOURCE_DIR}/${translation}")
	endforeach()
	add_custom_command(OUTPUT text-files.stamp
	    desktop_file_comment.txt mime_type_comment.txt
	  COMMAND "${CMAKE_COMMAND}" -E make_directory "../packaging/linux"
	  COMMAND "${CMAKE_COMMAND}" -E copy "${PROJECT_SOURCE_DIR}/packaging/linux/Mapper.desktop" "../packaging/linux/"
	  COMMAND "${CMAKE_COMMAND}" -E copy "${PROJECT_SOURCE_DIR}/packaging/linux/openorienteering-mapper.xml" "../packaging/linux/"
	  COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/translate_text_files.sh" ${translations}
	  ${update_sources}
	  COMMAND "${CMAKE_COMMAND}" -E touch text-files.stamp
	  DEPENDS translate_text_files.sh ${translations}
	  WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
	)
	add_custom_target(translate-text-files
	  DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/text-files.stamp"
	)
	add_dependencies(Mapper_translations translate-text-files)
endif()
