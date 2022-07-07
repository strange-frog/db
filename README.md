# C++简易实现数据库

 参考网上的数据库开发资料整理而成，源代码在window上编译成功。
 git地址：
 可执行文件：

 输入以下语句进行测试

 ```

 ```

 ```

 ```



## 实现REPL

REPL(Read-Eval-Print Loop，简称REPL) “读取-求值-输出”循环，
也被称做交互式顶层构件 _**(英語：interactive toplevel)**_，
是一个简单的交互式的编程环境。
  
 
##   SQL的解析前端
它将传统输入的`string`字符串，解析成可被机器识别的字节码内部表现形式，并传递给虚拟机进一步执行。
以`.`开头的非sql语句称作元命令 ***(meta command)*** 所以我们在一开始就检查是否以其开头，
并单独封装一个`do_meta_command`函数来处理它。
```c++
bool DB::parse_meta_command(std::string command)
{
    if (command[0] == '.')
    {
        switch (do_meta_command(command))
        {
        case META_COMMAND_SUCCESS:
            return true;
        case META_COMMAND_UNRECOGNIZED_COMMAND:
            std::cout << "Unrecognized command: " << command << std::endl;
            return true;
        }
    }
    return false;
}
MetaCommandResult DB::do_meta_command(std::string command)
{
    if (command == ".exit")
    {
        std::cout << "Bye!" << std::endl;
        exit(EXIT_SUCCESS);
    }
    else
    {
        return META_COMMAND_UNRECOGNIZED_COMMAND;
    }
}
```

 

一次性定义另外两个解析结果用于解析`sql`语句。
`    
enum PrepareResult
    {
        PREPARE_SUCCESS,
        PREPARE_UNRECOGNIZED_STATEMENT
    };
`
这个枚举结果用于返回我们所传递的`sql`语句是否是符合标准的状态。
这个时候就能将传统的以`select`或`insert`开头的`sql`语句转化成对应的
`statement`状态字节码。
```c++
PrepareResult DB::prepare_statement(std::string &input_line, Statement &statement)
{
    if (!input_line.compare(0, 6, "insert"))
    {
        statement.type = STATEMENT_INSERT;
        return PREPARE_SUCCESS;
    }
    else if (!input_line.compare(0, 6, "select"))
    {
        statement.type = STATEMENT_SELECT;
        return PREPARE_SUCCESS;
    }
    else
    {
        return PREPARE_UNRECOGNIZED_STATEMENT;
    }
}
```
我们为`sql`语句目前仅定义了如下简单的两种状态字节码
`
    enum StatementType
    {
        STATEMENT_INSERT,
        STATEMENT_SELECT
    };
`
同时再将上一步成功转化后得到的`statement`交给虚拟机进行解析。
```c++
bool DB::parse_statement(std::string &input_line, Statement &statement)
{
    switch (prepare_statement(input_line, statement))
    {
    case PREPARE_SUCCESS:
        return false;
    case PREPARE_UNRECOGNIZED_STATEMENT:
        std::cout << "Unrecognized keyword at start of '" << input_line << "'." << std::endl;
        return true;
    }
    return false;
}
```
至此我们初步完成了解析前端的工作，根据`command`或者`sql`得到了我们所需要的`statement`。

##  实现一个虚拟机
我们先根据得到的`statement`让虚拟机伪执行一下对应`sql`语句的操作效果。
```c++
void DB::excute_statement(Statement &statement)
{
    switch (statement.type)
    {
    case STATEMENT_INSERT:
        std::cout << "Executing insert statement" << std::endl;
        break;
    case STATEMENT_SELECT:
        std::cout << "Executing select statement" << std::endl;
        break;
    }
}
void DB::start()
{
    while (true)
    {
        print_prompt();

        std::string input_line;
        std::getline(std::cin, input_line);

        if (parse_meta_command(input_line))
        {
            continue;
        }

        Statement statement;

        if (parse_statement(input_line, statement))
        {
            continue;
        }

        execute_statement(statement);
    }
}
```

##  设计储存结构

我们目前规定所储存的类型结构如下
| 列       | 类型                    |
| -------- | ----------------------- |
| id       | 整型 ***(integer)***           |
| username | 可变字符串 ***(varchar 32)***  |
| email    | 可变字符串 ***(varchar 255)*** |

注意到我们这个时候没有使用 `std::string` 而使用的是 `char[]` 以限制长度。
```c++
#define COLUMN_USERNAME_SIZE 32
#define COLUMN_EMAIL_SIZE 255
class Row
{
public:

    uint32_t id;
    char username[COLUMN_USERNAME_SIZE];
    char email[COLUMN_EMAIL_SIZE];

}; 
```
现在我们已经有了一行行的数据了，现在让我们将它合理的储存起来。 
`B-Tree` 进行储存，这种优越的数据结构可以快速的进行查找，插入，删除。
但是显然我们应该从最简单的开始，我们选择将它分组到 `页面` ***(Page)***
当中去，然后将这些页面以数组的形式排列。此外，我们需要在一页中，尽可能的将其紧密排列，意味着数据应该一个挨着一个。
|列	|大小 ***(bytes)***|偏移量 ***(offset)***|
| -------- | ---- |---|
|id        |  4   |	0|
|username  |  32  |	4|
|email	   |  255 |36|
|总计	   |  291|  |
 
我们通过实现序列化 ***(serialize)*** 
以及反序列化 ***(serialize)*** 来达成我们的目的。
同时注意到我们这里写了一个`(char *)`的强制转化类型，是为了让编译器明白，
偏移量 ***(offset)*** 是以单个字节 ***(bytes)*** 为单位的。
```c++
#define size_of_attribute(Struct, Attribute) sizeof(((Struct *)0)->Attribute)

const uint32_t ID_SIZE = size_of_attribute(Row, id);
const uint32_t USERNAME_SIZE = size_of_attribute(Row, username);
const uint32_t EMAIL_SIZE = size_of_attribute(Row, email);
const uint32_t ID_OFFSET = 0;
const uint32_t USERNAME_OFFSET = ID_OFFSET + ID_SIZE;
const uint32_t EMAIL_OFFSET = USERNAME_OFFSET + USERNAME_SIZE;
const uint32_t ROW_SIZE = ID_SIZE + USERNAME_SIZE + EMAIL_SIZE;

void serialize_row(Row &source, void *destination)
{
    memcpy((char *)destination + ID_OFFSET, &(source.id), ID_SIZE);
    memcpy((char *)destination + USERNAME_OFFSET, &(source.username), USERNAME_SIZE);
    memcpy((char *)destination + EMAIL_OFFSET, &(source.email), EMAIL_SIZE);
}

void deserialize_row(void *source, Row &destination)
{
    memcpy(&(destination.id), (char *)source + ID_OFFSET, ID_SIZE);
    memcpy(&(destination.username), (char *)source + USERNAME_OFFSET, USERNAME_SIZE);
    memcpy(&(destination.email), (char *)source + EMAIL_OFFSET, EMAIL_SIZE);
}
```
接下来，让我们创建我们的`Table`来储存这些分页。同时我们与大多数计算机系统一样，将其设置为4k大小。
```c++
#define TABLE_MAX_PAGES 100
const uint32_t PAGE_SIZE = 4096;
const uint32_t ROWS_PER_PAGE = PAGE_SIZE / ROW_SIZE;
const uint32_t TABLE_MAX_ROWS = ROWS_PER_PAGE * TABLE_MAX_PAGES;
class Table
{
public:
    uint32_t num_rows;
    void *pages[TABLE_MAX_PAGES];
    Table()
    {
        num_rows = 0;
        for (uint32_t i = 0; i < TABLE_MAX_PAGES; i++)
        {
            pages[i] = NULL;
        }
    }
    ~Table()
    {
        for (int i = 0; pages[i]; i++)
        {
            free(pages[i]);
        }
    }
};
```
注意到，由于`C++`的构造和析构特性，使得我们在一定程度上减少了很多手动初始化
和释放的代码工作量。

此外我们还应该知道，我们的表页该从何处开始读写。
```c++
void *row_slot(Table &table, uint32_t row_num)
{
    uint32_t page_num = row_num / ROWS_PER_PAGE;
    void *page = table.pages[page_num];
    if (page == NULL)
    {
        // Allocate memory only when we try to access page
        page = table.pages[page_num] = malloc(PAGE_SIZE);
    }
    uint32_t row_offset = row_num % ROWS_PER_PAGE;
    uint32_t byte_offset = row_offset * ROW_SIZE;
    return (char *)page + byte_offset;
}
```

##  怎么实现执行储存操作
现在我们已经设计好了储存结构，让我们来看一下怎么执行储存操作。首先，我们为`statement`添加我们的`row`属性。
```c++
class Statement
{
public:
    StatementType type;
    Row row_to_insert;
};
```
之后我们需要解析`insert`这个操作。为此，我们对添加`PrepareResult`新增一个解析句法错误状态`PREPARE_SYNTAX_ERROR`。我们使用`sscanf`来进行格式化读取输入，并将它储存到我们的`statement`当中去。
```c++
PrepareResult DB::prepare_statement(std::string &input_line, Statement &statement)
{
    if (!input_line.compare(0, 6, "insert"))
    {
        statement.type = STATEMENT_INSERT;
        int args_assigned = std::sscanf(
            input_line.c_str(), "insert %d %s %s", &(statement.row_to_insert.id),
            statement.row_to_insert.username, statement.row_to_insert.email);
        if (args_assigned < 3)
        {
            return PREPARE_SYNTAX_ERROR;
        }
        return PREPARE_SUCCESS;
    }
    else if (!input_line.compare(0, 6, "select"))
    {
        statement.type = STATEMENT_SELECT;
        return PREPARE_SUCCESS;
    }
    else
    {
        return PREPARE_UNRECOGNIZED_STATEMENT;
    }
}
```
现在让我们来真正设计执行`insert`与`select`操作。

对于`insert`操作，我们首先判断它是否超出储存限制，针对操作执行结果，我们同样添加了与我们之前类似的枚举类状态码
`enum ExecuteResult
{
    EXECUTE_SUCCESS,
    EXECUTE_TABLE_FULL
};`
之后，若满足对应条件，我们寻找到合适的内存插入位置，将我们输入的行以`serialize_row`的方式填充到内存`page`当中。
```c++
ExecuteResult DB::execute_insert(Statement &statement, Table &table)
{
    if (table.num_rows >= TABLE_MAX_ROWS)
    {
        std::cout << "Error: Table full." << std::endl;
        return EXECUTE_TABLE_FULL;
    }

    void *page = row_slot(table, table.num_rows);
    serialize_row(statement.row_to_insert, page);
    table.num_rows++;

    return EXECUTE_SUCCESS;
}
```
类似的对于`select`操作，我们仅需从`page`中对应位置通过`deserialize_row`的方式获取到即可。
```c++
ExecuteResult DB::execute_select(Statement &statement, Table &table)
{
    for (uint32_t i = 0; i < table.num_rows; i++)
    {
        Row row;
        void *page = row_slot(table, i);
        deserialize_row(page, row);
        std::cout << "(" << row.id << ", " << row.username << ", " << row.email << ")" << std::endl;
    }

    return EXECUTE_SUCCESS;
}
```
最后将我们所设计好的操作交给`虚拟机`来执行即可。
```c++
void DB::execute_statement(Statement &statement, Table &table)
{
    ExecuteResult result;
    switch (statement.type)
    {
    case STATEMENT_INSERT:
        result = execute_insert(statement, table);
        break;
    case STATEMENT_SELECT:
        result = execute_select(statement, table);
        break;
    }

    switch (result)
    {
    case EXECUTE_SUCCESS:
        std::cout << "Executed." << std::endl;
        break;
    case EXECUTE_TABLE_FULL:
        std::cout << "Error: Table full." << std::endl;
        break;
    }
}

void DB::start()
{
    Table table;

    while (true)
    {
        print_prompt();

        std::string input_line;
        std::getline(std::cin, input_line);

        if (parse_meta_command(input_line))
        {
            continue;
        }

        Statement statement;

        if (parse_statement(input_line, statement))
        {
            continue;
        }

        execute_statement(statement, table);
    }
}
```

##  如何将数据储存到硬盘中？

首先，我们对`main`函数进行全新的处理，以便获得命令行参数。
```c++
int main(int argc, char const *argv[])
{
    if (argc < 2)
    {
        std::cout << "Must supply a database filename." << std::endl;
        exit(EXIT_FAILURE);
    }

    DB db(argv[1]);
    db.start();
    return 0;
}
```
我们看到，我们创建了一个全新的构造函数形式，以便我们可以接受命令行参数。我们接着往下看，我们将`Table`这个对象作为属性添加到了`DB`类中。这样子，我们的`execute`类函数不再需要额外的`table`参数传递，而是直接调用自身属性内的`table`对象。
```c++
class DB
{
private:
    Table* table;

public:
    DB(const char *filename)
    {
        table = new Table(filename);
    }
    void start();
    void print_prompt();

    bool parse_meta_command(std::string &command);
    MetaCommandResult do_meta_command(std::string &command);

    PrepareResult prepare_insert(std::string &input_line, Statement &statement);
    PrepareResult prepare_statement(std::string &input_line, Statement &statement);
    bool parse_statement(std::string &input_line, Statement &statement);
    void execute_statement(Statement &statement);
    ExecuteResult execute_insert(Statement &statement);
    ExecuteResult execute_select(Statement &statement);

    ~DB()
    {
        delete table;
    }
};
```
现在让我们来看一下我们在`new Table(filename)`中到底做了什么。
```c++
class Table
{
public:
    uint32_t num_rows;
    Pager pager;
    Table(const char *filename) : pager(filename)
    {
        num_rows = pager.file_length / ROW_SIZE;
    }
    ~Table();
    void *row_slot(uint32_t row_num);
};
```
我们创建了一个全新的`Pager分页`对象。那么这个`Pager`究竟是做了些什么呢？

##  如何实现一个分页?
现在我们见到了在`Table`中消失的`void *pages[TABLE_MAX_PAGES];`
```c++
class Pager
{
public:
    int file_descriptor;
    uint32_t file_length;
    void *pages[TABLE_MAX_PAGES];
    Pager(const char *filename);
    void *get_page(uint32_t page_num);
    void pager_flush(uint32_t page_num, uint32_t size);
};
```
我们首先来看一下如何构造这个`Pager`对象。
```c++
Pager::Pager(const char *filename)
{
    file_descriptor = open(filename,
                           O_RDWR |     // Read/Write mode
                               O_CREAT, // Create file if it does not exist
                           S_IWUSR |    // User write permission
                               S_IRUSR  // User read permission
    );
    if (file_descriptor < 0)
    {
        std::cerr << "Error: cannot open file " << filename << std::endl;
        exit(EXIT_FAILURE);
    }
    file_length = lseek(file_descriptor, 0, SEEK_END);

    for (uint32_t i = 0; i < TABLE_MAX_PAGES; i++)
    {
        pages[i] = nullptr;
    }
}
```
我们看到，我们创建了一个新的`file_descriptor`用作我们物理磁盘上储存交互，并且设置了`file_length`属性来获取其文件大小。

此外我们在这当中添加了一个`get_page`函数来作用于`row_slot`当中，用于获取指定页的内存。逻辑依旧十分简单，如果我们没有获取到页面，我们就创建一个新的页面，并且将其存储到`pages`数组中。
```c++
void *Table::row_slot(uint32_t row_num)
{
    uint32_t page_num = row_num / ROWS_PER_PAGE;
    void *page = pager.get_page(page_num);
    uint32_t row_offset = row_num % ROWS_PER_PAGE;
    uint32_t byte_offset = row_offset * ROW_SIZE;
    return (char *)page + byte_offset;
}

void *Pager::get_page(uint32_t page_num)
{
    if (page_num > TABLE_MAX_PAGES)
    {
        std::cout << "Tried to fetch page number out of bounds. " << page_num << " > "
                  << TABLE_MAX_PAGES << std::endl;
        exit(EXIT_FAILURE);
    }

    if (pages[page_num] == nullptr)
    {
        // Cache miss. Allocate memory and load from file.
        void *page = malloc(PAGE_SIZE);
        uint32_t num_pages = file_length / PAGE_SIZE;

        // We might save a partial page at the end of the file
        if (file_length % PAGE_SIZE)
        {
            num_pages += 1;
        }

        if (page_num <= num_pages)
        {
            lseek(file_descriptor, page_num * PAGE_SIZE, SEEK_SET);
            ssize_t bytes_read = read(file_descriptor, page, PAGE_SIZE);
            if (bytes_read == -1)
            {
                std::cout << "Error reading file: " << errno << std::endl;
                exit(EXIT_FAILURE);
            }
        }

        pages[page_num] = page;
    }

    return pages[page_num];
}
```
我们注意到，如果我们所获取的页码是已经在我们磁盘(文件)中的，我们就直接从其中读取。

## 4. 如何将内存中的数据写入磁盘?

关键行为规范的是，在我们的用户合理使用`.exit`退出时，我们便将所有的数据写入磁盘。
```c++
MetaCommandResult DB::do_meta_command(std::string &command)
{
    if (command == ".exit")
    {
        delete(table);
        std::cout << "Bye!" << std::endl;
        exit(EXIT_SUCCESS);
    }
    else
    {
        return META_COMMAND_UNRECOGNIZED_COMMAND;
    }
}
```
核心非常简单，就是`delete(table)`，这个函数将所有的数据写入磁盘。那让我们来看看它到底是怎么做的。
```c++
Table::~Table()
{
    uint32_t num_full_pages = num_rows / ROWS_PER_PAGE;

    for (uint32_t i = 0; i < num_full_pages; i++)
    {
        if (pager.pages[i] == nullptr)
        {
            continue;
        }
        pager.pager_flush(i, PAGE_SIZE);
        free(pager.pages[i]);
        pager.pages[i] = nullptr;
    }

    // There may be a partial page to write to the end of the file
    // This should not be needed after we switch to a B-tree
    uint32_t num_additional_rows = num_rows % ROWS_PER_PAGE;
    if (num_additional_rows > 0)
    {
        uint32_t page_num = num_full_pages;
        if (pager.pages[page_num] != nullptr)
        {
            pager.pager_flush(page_num, num_additional_rows * ROW_SIZE);
            free(pager.pages[page_num]);
            pager.pages[page_num] = nullptr;
        }
    }

    int result = close(pager.file_descriptor);
    if (result == -1)
    {
        std::cout << "Error closing db file." << std::endl;
        exit(EXIT_FAILURE);
    }
    for (uint32_t i = 0; i < TABLE_MAX_PAGES; i++)
    {
        void *page = pager.pages[i];
        if (page)
        {
            free(page);
            pager.pages[i] = nullptr;
        }
    }
}
```
在这个函数中，我们首先关闭文件，然后释放所有的页面。关键在于释放页面的前提是，我们调用了`page_flush`这个函数，先让我们来看看这个函数。
```c++
void Pager::pager_flush(uint32_t page_num, uint32_t size)
{
    if (pages[page_num] == nullptr)
    {
        std::cout << "Tried to flush null page" << std::endl;
        exit(EXIT_FAILURE);
    }

    off_t offset = lseek(file_descriptor, page_num * PAGE_SIZE, SEEK_SET);

    if (offset == -1)
    {
        std::cout << "Error seeking: " << errno << std::endl;
        exit(EXIT_FAILURE);
    }

    ssize_t bytes_written =
        write(file_descriptor, pages[page_num], size);

    if (bytes_written == -1)
    {
        std::cout << "Error writing: " << errno << std::endl;
        exit(EXIT_FAILURE);
    }
}
```
本质上来说非常简单，打开对应位置，借助`write`写入即可。

回到`~Table`的实现，我们可以看到，存在某种特殊情况，即单个页面中并没有全部使用，此时我们仅需要通过调整`flush`的`size`参数将剩余的部分写入磁盘即可。最后关闭文件并二次保险确认释放页面。

让我们来测试一下储存功能。
```
.......

Finished in 0.03502 seconds (files took 0.07738 seconds to load)
7 examples, 0 failures
```
 


## 1. 我们的光标需要做什么？
* 指向`table`开头
* 指向`table`结尾
* 指向`table`中的一个`row`
* 将`cursor`往后推进

## 2. 如何实现光标？
显然我们现在要指向`table`开头/结尾，所以我们需要实现一个`cursor`，它可以指向`table`开头，也可以指向`table`结尾。注意我们即然使用了`cursor`，也即`指向`这个词，我们在此处使用的就是`指针`，使用的其实一直就是`DB::table`唯一对象。
```c++
class Cursor
{
public:
    Table *table;
    uint32_t row_num;
    bool end_of_table;

    Cursor(Table *&table, bool option);
    void *cursor_value();
    void cursor_advance();
};
Cursor::Cursor(Table *&table, bool option)
{
    this->table = table;
    if (option)
    {
        // start at the beginning of the table
        row_num = 0;
        end_of_table = (table->num_rows == 0);
    }
    else
    {
        // end of the table
        row_num = table->num_rows;
        end_of_table = true;
    }
}
```
同时我们将不再使用`row_slot`这个函数，而转为使用`cursor_value`函数。
```c++
void *Cursor::cursor_value()
{
    uint32_t page_num = row_num / ROWS_PER_PAGE;
    void *page = table->pager.get_page(page_num);
    uint32_t row_offset = row_num % ROWS_PER_PAGE;
    uint32_t byte_offset = row_offset * ROW_SIZE;
    return (char *)page + byte_offset;
}
```
注意到，其实基本没有任何区别。

现在让我们来看看`cursor_advance`函数，它的作用是将`cursor`往后推进一个`row`。
```c++
void Cursor::cursor_advance()
{
    row_num += 1;
    if (row_num >= table->num_rows)
    {
        end_of_table = true;
    }
}
```

## 3. 如何使用我们的光标？
现在让我们来看如何使用光标进行`insert`操作。我们先创建一个指向`table`末尾的`cursor`，然后调用`cursor_value`函数便可直接获得对应分页信息。关键点其实，我们本质上调用的都是同一个`table`的`pager`的`get_page`函数，而`pager`的`get_page`函数的作用是从`pager`中获取分页信息。
```c++
ExecuteResult DB::execute_insert(Statement &statement)
{
    if (table->num_rows >= TABLE_MAX_ROWS)
    {
        std::cout << "Error: Table full." << std::endl;
        return EXECUTE_TABLE_FULL;
    }

    // end of the table
    Cursor *cursor = new Cursor(table, false);

    serialize_row(statement.row_to_insert, cursor->cursor_value());
    table->num_rows++;

    delete cursor;

    return EXECUTE_SUCCESS;
}
```
再看一下`select`操作，我们先创建一个指向`table`开头的`cursor`，然后同样调用`cursor_value`函数便可直接获得对应分页信息。再使用`cursor_advance`函数，将`cursor`往后推进一个`row`。
```c++
ExecuteResult DB::execute_select(Statement &statement)
{
    // start of the table
    Cursor *cursor = new Cursor(table, true);

    Row row;
    while (!cursor->end_of_table)
    {
        deserialize_row(cursor->cursor_value(), row);
        std::cout << "(" << row.id << ", " << row.username << ", " << row.email << ")" << std::endl;
        cursor->cursor_advance();
    }

    delete cursor;

    return EXECUTE_SUCCESS;
}
```

## 1. B-Tree是什么？
B树 ***（B-tree）*** 是一种自平衡的树，能够保持数据有序。这种资料结构能够让查找数据、顺序访问、插入数据及删除的动作，都在对数时间内完成。 B树，概括来说是一个一般化的二元搜寻树（binary search tree）一个结点可以拥有2个以上的子结点。与自平衡二叉查找树不同，B树适用于读写相对大的数据块的存储系统，例如磁盘。
 B树减少定位记录时所经历的中间过程，从而加快存取速度。 B树这种数据结构可以用来描述外部存储。这种资料结构常被应用在数据库和文件系统的实现上。
![example B-Tree](https://upload.wikimedia.org/wikipedia/commons/6/65/B-tree.svg)
与二叉树不同，B-Tree中的每个结点可以有超过2个子结点。每个结点最多可以有m子结点，其中m称为树的“阶”。为了保持树的大部分平衡，我们还说结点必须至少有m/2子结点（四舍五入）。

但实际上，我们在这里使用的是B-Tree的一个变种情况，即：B+Tree。
我们在其中储存我们的数据，并且每个结点中存在多个键值对，而且这个键值是按照顺序排列的。

使用这种结构我们的查找时间复杂度是**O(log(n))**，而且插入和删除的时间复杂度也是**O(log(n))**。

## 2. 如何实现B-Tree?


按照惯例，我们先增加一个测试用例，对应的已经是 `3-leaf-node btree` 。


当叶子结点满了之后，我们需要对其进行分裂，这个分裂标准是什么？

我们需要将现有的 `cell` 分成两个部分：上半部分与下半部分。
上半部分的 `key` 严格大于下半部分的 `key` ，这样我们就可以将 `cell` 
分成两个部分。因此，我们分配一个新的叶子结点，并将对应的上半部分移入
该叶子结点。
按照惯例，先看一下假设我们修复错误后应该如何正常打印我们的`B-Tree`。
核心仍旧是使用二分搜索+递归。我们知道内部结点中所储存的孩子结点的指针的右侧储存的是该孩子指针所包含的最大的`key`值，所以我们只需要将被搜索的`key`不断与`key_to_right`进行比较，直到找到`key`的位置。

此外，当我们找到了对应的孩子结点时，要注意判断其类型仍旧为`InternalNode`，我们需要递归调用`internal_node_find`。亦或是我们找到了`LeafNode`，我们仅需要返回一个相应指向该结点的`Cursor`对象即可。

为了保证我们能够在打印到第一个叶子结点的末端时，自动跳转到第二个叶子结点,我们在其标头设置相应的`next_leaf`字段来指向下一个叶子结点。
## 1. 为什么要更新拆分后的父结点？
## 2. 如何实现更新拆分后的父结点？