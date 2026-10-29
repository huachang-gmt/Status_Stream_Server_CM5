cmake_minimum_required(VERSION 3.16)

project(Status_Stream_Server_CM5 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_executable(Status_Stream_Server_CM5
    src/main.cpp
)