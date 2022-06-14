cmake_minimum_required(VERSION 3.18.1)

project("cxx-provider")

add_library(
        cxx-provider
        SHARED
        package-info.cpp)

target_link_libraries(
        cxx-provider
        )
