set(TEST_FILES_SHARED
    cagg_compression.sql
    chunkwise_agg_gather_sort.sql
    classify_relation.sql
    compat.sql
    compress_bloom_sparse_compat.sql
    compress_unique_index.sql
    compression_dml.sql
    compression_nulls_not_distinct.sql
    constify_timestamptz_op_interval.sql
    constraint_aware_append.sql
    constraint_exclusion_prepared.sql
    decompress_join.sql
    decompress_placeholdervar.sql
    decompress_tracking.sql
    gapfill_bug.sql
    generated_columns.sql
    memoize.sql
    security_barrier.sql
    space_constraint.sql
    subtract_integer_from_now.sql)

set(TEST_TEMPLATES_SHARED gapfill.sql.in ordered_append_join.sql.in
                          transparent_decompress_chunk.sql.in)

set(SOLO_TESTS merge_dml.sql)

if(CMAKE_BUILD_TYPE MATCHES Debug)
  list(APPEND TEST_FILES_SHARED extension.sql timestamp_limits.sql
       with_clause_parser.sql)
  list(APPEND TEST_FILES_SHARED constify_now.sql)
endif(CMAKE_BUILD_TYPE MATCHES Debug)

# Regression tests that vary with PostgreSQL version. Generated test files are
# put in the original source directory since all tests must be in the same
# directory. These files are updated when the template is edited, but not when
# the output file is deleted. If the output is deleted either recreate it
# manually, or rerun cmake on the root dir.
foreach(TEMPLATE_FILE ${TEST_TEMPLATES_SHARED})
  string(LENGTH ${TEMPLATE_FILE} TEMPLATE_NAME_LEN)
  math(EXPR TEMPLATE_NAME_LEN ${TEMPLATE_NAME_LEN}-7)
  string(SUBSTRING ${TEMPLATE_FILE} 0 ${TEMPLATE_NAME_LEN} TEMPLATE)
  set(TEST_FILE ${TEMPLATE}-${TEST_VERSION_SUFFIX}.sql)
  configure_file(${TEMPLATE_FILE} ${CMAKE_CURRENT_SOURCE_DIR}/${TEST_FILE}
                 COPYONLY)
  list(APPEND TEST_FILES_SHARED ${TEST_FILE})
endforeach(TEMPLATE_FILE)

if(DEFINED GIT_FOUND)
  list(APPEND TEST_FILES_SHARED build_info.sql)
endif()

if(TEST_FILES_SHARED)
  list(SORT TEST_FILES_SHARED)
endif()
file(REMOVE ${TEST_SCHEDULE_SHARED})

if(NOT TEST_GROUP_SIZE)
  set(PARALLEL_GROUP_SIZE 20)
else()
  set(PARALLEL_GROUP_SIZE ${TEST_GROUP_SIZE})
endif()

set(GROUP_MEMBERS 0)
foreach(TEST_FILE ${TEST_FILES_SHARED})
  string(REGEX REPLACE "(.+)\.sql" "\\1" TESTS_TO_RUN ${TEST_FILE})
  if(GROUP_MEMBERS EQUAL 0)
    file(APPEND ${TEST_SCHEDULE_SHARED} "\ntest: ")
  endif()
  file(APPEND ${TEST_SCHEDULE_SHARED} "${TESTS_TO_RUN} ")
  math(EXPR GROUP_MEMBERS "(${GROUP_MEMBERS}+1)%${PARALLEL_GROUP_SIZE}")
endforeach(TEST_FILE)
file(APPEND ${TEST_SCHEDULE_SHARED} "\n")

set(GROUP_MEMBERS 0)
foreach(TEST_FILE ${SOLO_TESTS})
  string(REGEX REPLACE "(.+)\.sql" "\\1" TESTS_TO_RUN ${TEST_FILE})
  if(GROUP_MEMBERS EQUAL 0)
    file(APPEND ${TEST_SCHEDULE_SHARED} "\ntest: ")
  endif()
  file(APPEND ${TEST_SCHEDULE_SHARED} "${TESTS_TO_RUN} ")
  math(EXPR GROUP_MEMBERS "(${GROUP_MEMBERS}+1)%${PARALLEL_GROUP_SIZE}")
endforeach(TEST_FILE)
file(APPEND ${TEST_SCHEDULE_SHARED} "\n")
