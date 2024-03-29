TLDR: Код на C++ можно разбивать на единицы трансляции, собирающиеся независимо, для этого существует механизм внутренних и внешних связей, называемый линковкой. Это позволяет создавать исполняемые файлы, статические и динамические библиотеки. Для удобной работы с большим количеством сущностей существуют системы сборки, например make и cmake.

[//]:<>(preview_end)

## Что это такое и зачем нужно?

Рано или поздно (чаще рано) любой проект приходит в состояние, когда весь код не умещается в один файл. На это может быть сразу несколько причин: логических сущностей становится все больше, намечается явное разделение на модули, строчек кода становится слишком много. Тогда интуитивным решением будет разделить кодовую базу на несколько файлов, объединив код в них по некоторой логике.

Следующим закономерным шагом будет не только формальное разделение на файлы, но и на отдельные таргеты компиляции, называемые *единицами трансляции*. Для удобства понимания можно считать, что это модули - атомарные подпроекты, содержащие свою более-менее независимую логику. После компиляции каждой такой сущности образуется бинарный артефакт, который можно *прилинковать* к остальным и получить конечный исполняемый файл или библиотеку.

Такое разделение на уровне компиляции дает основное преимущество модульной архитектуры по сравнению с монолитной - теперь проект можно компилировать независимо по единицам трансляции. При изменении кода, не нужно пересобирать всю кодовую базу: достаточно пересобрать единицу трансляции, к которой он относится. Это существенно экономит время сборки при разработке всего проекта.

## Как это работает в коде?

Скомпилируем простейшую программу, не использующую сторонних библиотек.

```
int main() {
	return 1;
}
```
В директории появился файл **a.out**. Это бинарный файл в формате [ELF (Executable and Linkable Format)](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format).  Для его анализа существует инструмент **readelf** - он позволяет посмотреть все низкоуровневые сущности. Однако в рамках этой статьи нас будут интересовать только сами символы. Утилита **mcview** умеет удобно парсить elf и показывать их список:

![](https://habrastorage.org/webt/gf/zf/3n/gfzf3ngekobjqrcvu5h4pveyv6m.png)Большая часть скомпилированного файла состоит из служебных символов библиотеки [**glibc**](https://en.wikipedia.org/wiki/GNU_C_Library), обрабатывающих системные вызовы. Нас интересует только **main**.

Теперь попробуем добавить в программу функцию, и сразу увидим, что в elf артефакте появился новый символ с ее названием:

```
double average(int a, int b) {
	return static_cast<double>(a + b) / 2.0;
}
int main() {
	int a = 10;
	int b = 20;
	return static_cast<int>(average(a, b));
}
```
![](https://habrastorage.org/webt/ve/tm/vp/vetmvpbfoox4lsqypk6x9obrvmi.png)

Далее попробуем вынести эту функцию в отдельную единицу трансляции. Для этого создадим рядом файл average.cpp с ней:
```
double average(int a, int b) {
	return  static_cast<double>(a + b) / 2.0;
}
```
А в main.cpp уберем определение функции и оставим только объявление:
```
double average(int a, int b);

int main() {
	int a = 10;
	int b = 20;
	return  static_cast<int>(average(a, b));
}
```

Теперь если собрать average.cpp как <!объектный файл!>(Какой командой?){g++ -c average.cpp}, образуется объектный файл average.o, который содержит только этот символ:
![](https://habrastorage.org/webt/dv/h9/d7/dvh9d7fqralg6ep3qmmf9nzvkui.png)

Наконец, мы можем <!собрать!>(Какой командой?){g++ main.cpp average.o} его вместе с основным main-ом в исполняемый файл.

## Статические и динамические библиотеки

Логичным развитием идеи обобщения и переиспользования кода будет распространение прекомпилированных объектных файлов для линковки их в сторонних проектах.

Для этого в C++ существует концепция статических и динамических библиотек. Начнем с первых.

Статические библиотеки - это все те же объектные файлы, сопровождаемые заголовочными .h файлами. Чтобы подключить такую библиотеку в коде, нужно импортировать заголовочный файл и при компиляции в качестве объектных файлов для линковщика указать объектный файл библиотеки.

Динамические библиотеки отличаются от статических тем, что использующий их код собирается без них, и вся библиотечная логика вызывается на стороне клиента уже при запуске.

## CMake

Способ сборки проектных таргетов через вызов нативных команд компилятора с указанием всего списка используемых объектных файлов и исходных кодов по <!очевидным причинам!>(Все-таки каким?){Для каждого файла нужно следить к какому таргету он относится и от каких зависит. Даже для проекта среднего размера держать эту неявную логику зависимостей в голове невозможно.} неудобен.

Для хранения информации о зависимостях между проектными сущностями и эффективной компиляции таргетов существуют различные системы сборки. Одной из самых распространенных, удобных и простых является **[CMake](https://cmake.org/cmake/help/latest/guide/tutorial/index.html)**.

Его базовый синтаксис довольно прост и основан на механизме уже упомянутых *таргетов* - единиц компиляции:
```
cmake_minimum_required(**VERSION** 2.8)
project(sample_4)
set(CMAKE_CXX_FLAGS
  "${CMAKE_CXX_FLAGS} -std=c++11 -Wall")

set(AVERAGE_SOURCE average.cpp)
add_library(AVERAGE average.cpp)

set(MAIN_SOURCE main.cpp)
add_executable(MAIN main.cpp)

target_link_libraries(MAIN AVERAGE)
```
В коде выше создается два таргета: **AVERAGE** - библиотека, содержащая функцию вычисления среднего значения и **MAIN** - собственно исполняемый файл, вызывающий эту функцию.

Последняя строчка обозначает зависимость **MAIN**  от **AVERAGE**.

Теперь чтобы скомпилировать проект достаточно запустить следующие команды:

```bash
mkdir build && cd build
cmake .. && make
```

Результат:

```txt
$ cmake .. && make
-- The C compiler identification is Clang 10.0.0
-- The CXX compiler identification is Clang 10.0.0
-- Check for working C compiler: /usr/bin/clang
-- Check for working C compiler: /usr/bin/clang -- works
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Detecting C compile features
-- Detecting C compile features - done
-- Check for working CXX compiler: /usr/bin/clang++
-- Check for working CXX compiler: /usr/bin/clang++ -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done
-- Generating done
-- Build files have been written to: /home/evgenstf/z_sort_sources/cpp/cpp_linking/sample_4/build
**Scanning dependencies of target AVERAGE**
[ 25%] Building CXX object CMakeFiles/AVERAGE.dir/average.cpp.o
[ 50%] **Linking CXX static library libAVERAGE.a**
[ 50%] Built target AVERAGE
**Scanning dependencies of target MAIN**
[ 75%] Building CXX object CMakeFiles/MAIN.dir/main.cpp.o
[100%] **Linking CXX executable MAIN**
[100%] Built target MAIN
```
После выполнения сборки в папке build появляются соответствующие артефакты **MAIN** и **libAVERAGE.a**.
<p style='text-align: right;'> *Весь представленный в статье код можно найти в [соответствующем репозитории](https://github.com/evgenstf/z_sort_sources/tree/main/cpp/cpp_linking).* </p>

