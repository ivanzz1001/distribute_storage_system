# protobuf反射

参看:

- [精通 protobuf 原理之三：一文彻底搞懂反射原理](https://zhuanlan.zhihu.com/p/665117667)

- [巧用 Protobuf 反射来优化代码，拒做 PB Boy](https://zhuanlan.zhihu.com/p/313369306)


- [protobuf官方网址](https://developers.google.com/protocol-buffers)

- [github源码](https://github.com/protocolbuffers/protobuf)

- [google protobuf 反射机制学习笔记](https://cchd0001.gitbooks.io/my-blog/content/protobufreflection.html)


## 1. protobuf反射机理

protobuf反射的基本原理就是利用编码过的`DescriptorTable`在内存中构建出相应的Descriptor， 然后MessageFactory使用该Descriptor来构建出具体的message对象实例。


本文结合目录中的echo_c++来讲述这一过程。首先我们来看一下DescriptorTable结构(src/google/protobuf/generated_message_reflection.h):

```
// This struct tries to reduce unnecessary padding.
// The num_xxx might not be close to their respective pointer, but this saves
// padding.
struct PROTOBUF_EXPORT DescriptorTable {
  mutable bool is_initialized;
  bool is_eager;
  int size;  // of serialized descriptor
  const char* descriptor;
  const char* filename;
  absl::once_flag* once;
  const DescriptorTable* const* deps;
  int num_deps;
  int num_messages;
  const MigrationSchema* schemas;
  const Message* const* default_instances;
  const uint32_t* offsets;
  // update the following descriptor arrays.
  Metadata* file_level_metadata;
  const EnumDescriptor** file_level_enum_descriptors;
  const ServiceDescriptor** file_level_service_descriptors;
};
```

这里简单讲述一下各个字段的含义：

- is_initialized: 用于指示当前DescriptorTable是否被初始化过

- is_eager: 当使用本DescriptorTable直接获取对应的MetaData时是否触发同时设置deps的MetaData

- size: 用于指示序列化的后的descriptor所占用的字节数

- descriptor: 序列化后的descriptor

- filename: 对应的proto文件名

- once: 配合std::call_once一起使用

- deps: 本DescriptorTable的依赖

- num_deps: deps的数量

- num_messages: 用于指示当前DescriptorTable中包含的message的数量

- schemas: 为一个实验特性，这里不讲述

- default_instances: 所对应的messages的默认实例

- offsets: 用于记录相关偏移

- file_level_metadata: proto文件级别的元数据，通常会在构建Descriptors时就会填充本字段

- file_level_enum_descriptors: proto文件级别的enum描述

- file_level_service_descriptors: proto文件级别的service描述

这里结合echo_c++示例，给出protoc对echo.proto编译后生成的DescriptorTable:

```
const ::_pbi::DescriptorTable descriptor_table_echo_2eproto = {
    false,
    false,
    159,
    descriptor_table_protodef_echo_2eproto,
    "echo.proto",
    &descriptor_table_echo_2eproto_once,
    nullptr,
    0,
    2,
    schemas,
    file_default_instances,
    TableStruct_echo_2eproto::offsets,
    file_level_metadata_echo_2eproto,
    file_level_enum_descriptors_echo_2eproto,
    file_level_service_descriptors_echo_2eproto,
};

const char descriptor_table_protodef_echo_2eproto[] PROTOBUF_SECTION_VARIABLE(protodesc_cold) = {
    "\n\necho.proto\022\007example\"\036\n\013EchoRequest\022\017\n\007"
    "message\030\001 \002(\t\"\037\n\014EchoResponse\022\017\n\007message"
    "\030\001 \002(\t2B\n\013EchoService\0223\n\004Echo\022\024.example."
    "EchoRequest\032\025.example.EchoResponseB\003\200\001\001"
};

static ::absl::once_flag descriptor_table_echo_2eproto_once;

static const ::_pbi::MigrationSchema
    schemas[] PROTOBUF_SECTION_VARIABLE(protodesc_cold) = {
        {0, 9, -1, sizeof(::example::EchoRequest)},
        {10, 19, -1, sizeof(::example::EchoResponse)},
};

static const ::_pb::Message* const file_default_instances[] = {
    &::example::_EchoRequest_default_instance_._instance,
    &::example::_EchoResponse_default_instance_._instance,
};

const ::uint32_t TableStruct_echo_2eproto::offsets[] PROTOBUF_SECTION_VARIABLE(
    protodesc_cold) = {
    PROTOBUF_FIELD_OFFSET(::example::EchoRequest, _impl_._has_bits_),
    PROTOBUF_FIELD_OFFSET(::example::EchoRequest, _internal_metadata_),
    ~0u,  // no _extensions_
    ~0u,  // no _oneof_case_
    ~0u,  // no _weak_field_map_
    ~0u,  // no _inlined_string_donated_
    ~0u,  // no _split_
    ~0u,  // no sizeof(Split)
    PROTOBUF_FIELD_OFFSET(::example::EchoRequest, _impl_.message_),
    0,
    PROTOBUF_FIELD_OFFSET(::example::EchoResponse, _impl_._has_bits_),
    PROTOBUF_FIELD_OFFSET(::example::EchoResponse, _internal_metadata_),
    ~0u,  // no _extensions_
    ~0u,  // no _oneof_case_
    ~0u,  // no _weak_field_map_
    ~0u,  // no _inlined_string_donated_
    ~0u,  // no _split_
    ~0u,  // no sizeof(Split)
    PROTOBUF_FIELD_OFFSET(::example::EchoResponse, _impl_.message_),
    0,
};

static ::_pb::Metadata file_level_metadata_echo_2eproto[2];

static constexpr const ::_pb::EnumDescriptor**
    file_level_enum_descriptors_echo_2eproto = nullptr;

static const ::_pb::ServiceDescriptor*
    file_level_service_descriptors_echo_2eproto[1];
```

>ps: 上面echo_2eproto的含义猜测应该是echo to encoded proto的含义


## 2. 基于DescriptorTable在DescriptorPool中构建出Descriptors

在echo_c++示例中，我们会发现有如下静态全局变量：

```
// Force running AddDescriptors() at dynamic initialization time.
PROTOBUF_ATTRIBUTE_INIT_PRIORITY2
static ::_pbi::AddDescriptorsRunner dynamic_init_dummy_echo_2eproto(&descriptor_table_echo_2eproto);
```

Descriptors的构建就是从该全局变量开始的。下面我们来分析一下这个过程。


### 2.1 AddDescriptorsRunner在构造函数中触发创建Descriptors

参看如下代码(src/google/protobuf/generated_message_reflection.cc)：

```
AddDescriptorsRunner::AddDescriptorsRunner(const DescriptorTable* table) {
  AddDescriptors(table);
}

void AddDescriptors(const DescriptorTable* table) {
  // AddDescriptors is not thread safe. Callers need to ensure calls are
  // properly serialized. This function is only called pre-main by global
  // descriptors and we can assume single threaded access or it's called
  // by AssignDescriptorImpl which uses a mutex to sequence calls.
  if (table->is_initialized) return;
  table->is_initialized = true;
  AddDescriptorsImpl(table);
}

void AddDescriptorsImpl(const DescriptorTable* table) {
  // Reflection refers to the default fields so make sure they are initialized.
  internal::InitProtobufDefaults();
  internal::InitializeFileDescriptorDefaultInstances();

  // Ensure all dependent descriptors are registered to the generated descriptor
  // pool and message factory.
  int num_deps = table->num_deps;
  for (int i = 0; i < num_deps; i++) {
    // In case of weak fields deps[i] could be null.
    if (table->deps[i]) AddDescriptors(table->deps[i]);
  }

  // Register the descriptor of this file.
  DescriptorPool::InternalAddGeneratedFile(table->descriptor, table->size);
  MessageFactory::InternalRegisterGeneratedFile(table);
}
```

从上面我们可以看到AddDescriptorsImpl()主要做了如下三件事：

- 初始化变量：反射需要使用的变量，先确保其已经初始化了

- 解析依赖：如果有import 其他proto 源文件，那么先解析其他proto源文件，存在 deps 中。

- 注册Descriptor

    - 构建 DescriptorPool 索引（DescriptorPool::InternalAddGeneratedFile）

    - 构建MessageFactory 索引（MessageFactory::InternalRegisterGeneratedFile）

## 3. 构建DescriptorPool索引


在具体讲解构建DescriptorPool索引之前，我们先简单介绍一下DescriptorPool数据结构(src/google/protobuf/descriptor.h)：

```
class PROTOBUF_EXPORT DescriptorPool {

	const FileDescriptor* FindFileByName(absl::string_view name) const;

	const FileDescriptor* FindFileContainingSymbol(
      absl::string_view symbol_name) const;

	const Descriptor* FindMessageTypeByName(absl::string_view name) const;
	
	const FieldDescriptor* FindFieldByName(absl::string_view name) const;
	const FieldDescriptor* FindExtensionByName(absl::string_view name) const;
	const OneofDescriptor* FindOneofByName(absl::string_view name) const;
	const EnumDescriptor* FindEnumTypeByName(absl::string_view name) const;
	const EnumValueDescriptor* FindEnumValueByName(absl::string_view name) const;
	const ServiceDescriptor* FindServiceByName(absl::string_view name) const;
	const MethodDescriptor* FindMethodByName(absl::string_view name) const;
};
```
很明显，因为我们的Descriptors都是通过DescriptorPool放进去的，因此其肯定提供了对应的方法来查询。


接下来我们看看DescriptorPool::InternalAddGeneratedFile()的实现(src/google/protobuf/descriptor.cc)：

```
void DescriptorPool::InternalAddGeneratedFile(
    const void* encoded_file_descriptor, int size) {
  // So, this function is called in the process of initializing the
  // descriptors for generated proto classes.  Each generated .pb.cc file
  // has an internal procedure called AddDescriptors() which is called at
  // process startup, and that function calls this one in order to register
  // the raw bytes of the FileDescriptorProto representing the file.
  //
  // We do not actually construct the descriptor objects right away.  We just
  // hang on to the bytes until they are actually needed.  We actually construct
  // the descriptor the first time one of the following things happens:
  // * Someone calls a method like descriptor(), GetDescriptor(), or
  //   GetReflection() on the generated types, which requires returning the
  //   descriptor or an object based on it.
  // * Someone looks up the descriptor in DescriptorPool::generated_pool().
  //
  // Once one of these happens, the DescriptorPool actually parses the
  // FileDescriptorProto and generates a FileDescriptor (and all its children)
  // based on it.
  //
  // Note that FileDescriptorProto is itself a generated protocol message.
  // Therefore, when we parse one, we have to be very careful to avoid using
  // any descriptor-based operations, since this might cause infinite recursion
  // or deadlock.
  absl::MutexLockMaybe lock(internal_generated_pool()->mutex_);
  ABSL_CHECK(GeneratedDatabase()->Add(encoded_file_descriptor, size));
}

EncodedDescriptorDatabase* GeneratedDatabase() {
  static auto generated_database =
      internal::OnShutdownDelete(new EncodedDescriptorDatabase());
  return generated_database;
}


```

从上面我们可以看到DescriptorPool实际是往一个全局静态的`EncodedDescriptorDatabase`中来添加encoded_file_descriptor的。database 是一个比较抽象的名称，其底层实现其实就是索引`index_`。下面我们来看其Add()方法实现:

```
bool EncodedDescriptorDatabase::Add(const void* encoded_file_descriptor,
                                    int size) {
  FileDescriptorProto file;
  if (file.ParseFromArray(encoded_file_descriptor, size)) {
    return index_->AddFile(file, std::make_pair(encoded_file_descriptor, size));
  } else {
    ABSL_LOG(ERROR) << "Invalid file descriptor data passed to "
                       "EncodedDescriptorDatabase::Add().";
    return false;
  }
}
```

首先把文件定义信息先解析，主要是确定 encoded_file_descriptor 信息是正确的。之用调用`index_`来进行添加。

### 3.1 DescriptorIndex介绍

EncodedDescriptorDatabase实例中有一个DescriptorIndex成员，其会在创建EncodedDescriptorDatabase对象时也一并创建出来。

下面我们简要介绍DescriptorIndex结构(src/google/protobuf/descriptor_database.cc):

```
class EncodedDescriptorDatabase::DescriptorIndex {
 	std::vector<EncodedEntry> all_values_;

	absl::btree_set<FileEntry, FileCompare> by_name_{FileCompare{*this}};
	std::vector<FileEntry> by_name_flat_;

	absl::btree_set<SymbolEntry, SymbolCompare> by_symbol_{SymbolCompare{*this}};
	std::vector<SymbolEntry> by_symbol_flat_;

	absl::btree_set<ExtensionEntry, ExtensionCompare> by_extension_{
      ExtensionCompare{*this}};
	std::vector<ExtensionEntry> by_extension_flat_;
};
```

其中：

- all_values_: 数据表，存的是最终从DescriptorTable读取到的已经被编码过的数据，包括文件元数据/符号元数据/扩展元数据

- by_name_flat_: 文件元数据 

- by_symbol_flat_: 符号元数据

- by_extension_flat_: 扩展元数据

### 3.2 向DescriptorIndex中添加encoded_file_descriptor


我们来看DescriptorIndex::Add()函数的实现：

```
template <typename FileProto>
bool EncodedDescriptorDatabase::DescriptorIndex::AddFile(const FileProto& file,
                                                         Value value) {
  // We push `value` into the array first. This is important because the AddXXX
  // functions below will expect it to be there.
  all_values_.push_back({value.first, value.second, {}});

  if (!ValidateSymbolName(file.package())) {
    ABSL_LOG(ERROR) << "Invalid package name: " << file.package();
    return false;
  }
  all_values_.back().encoded_package = EncodeString(file.package());


  //构建文件元信息索引
  if (!by_name_
           .insert({static_cast<int>(all_values_.size() - 1),
                    EncodeString(file.name())})
           .second ||
      std::binary_search(by_name_flat_.begin(), by_name_flat_.end(),
                         file.name(), by_name_.key_comp())) {
    ABSL_LOG(ERROR) << "File already exists in database: " << file.name();
    return false;
  }


  //构建symbol索引和扩展索引
  for (const auto& message_type : file.message_type()) {
    if (!AddSymbol(message_type.name())) return false;
    if (!AddNestedExtensions(file.name(), message_type)) return false;
  }
  for (const auto& enum_type : file.enum_type()) {
    if (!AddSymbol(enum_type.name())) return false;
  }
  for (const auto& extension : file.extension()) {
    if (!AddSymbol(extension.name())) return false;
    if (!AddExtension(file.name(), extension)) return false;
  }
  for (const auto& service : file.service()) {
    if (!AddSymbol(service.name())) return false;
  }

  return true;
}
```

讲到这里，构建索引的实现原理就告了一段落，但是你以为构建索引已经结束了吗？当然没有！在下一节中分析。

### 3.3 DescriptorPool索引的查询过程

前面我们讲到DescriptorPool提供了查询各种Descriptor的方法，这里我们以FindMessageTypeByName()为例来看一下其查询过程：

```
const Descriptor* DescriptorPool::FindMessageTypeByName(
    absl::string_view name) const {
  return tables_->FindByNameHelper(this, name).descriptor();
}

Symbol DescriptorPool::Tables::FindByNameHelper(const DescriptorPool* pool,
                                                absl::string_view name) {
  if (pool->mutex_ != nullptr) {
    // Fast path: the Symbol is already cached.  This is just a hash lookup.
    absl::ReaderMutexLock lock(pool->mutex_);
    if (known_bad_symbols_.empty() && known_bad_files_.empty()) {
      Symbol result = FindSymbol(name);
      if (!result.IsNull()) return result;
    }
  }
  absl::MutexLockMaybe lock(pool->mutex_);
  if (pool->fallback_database_ != nullptr) {
    known_bad_symbols_.clear();
    known_bad_files_.clear();
  }
  Symbol result = FindSymbol(name);

  if (result.IsNull() && pool->underlay_ != nullptr) {
    // Symbol not found; check the underlay.
    result = pool->underlay_->tables_->FindByNameHelper(pool->underlay_, name);
  }

  if (result.IsNull()) {
    // Symbol still not found, so check fallback database.
    if (pool->TryFindSymbolInFallbackDatabase(name)) {
      result = FindSymbol(name);
    }
  }

  return result;
}
```

FindByNameHelper 函数也并不复杂，首先查表，如果miss，就会调用 TryFindSymbolInFallbackDatabase 进行索引构建（这里会用到之前讲过的 DescriptorIndex 的信息）。

>ps: 这里刻意略过 underlay，underlay 这个特性笔者猜测是为了效率实现的多层cache，underlay 也就是下层的意思，逻辑都是一样的，这里我们没有涉及 underlay，就先不展开分析。


接着我们来看FindSymbol()函数：

```
inline Symbol DescriptorPool::Tables::FindSymbol(absl::string_view key) const {
  auto it = symbols_by_name_.find(FullNameQuery{key});
  return it == symbols_by_name_.end() ? Symbol() : *it;
}
```

我们发现其查的是 symbols_by_name_ 这个索引表。但是这个表我们还没有构建啊。是的，之前没有构建过，但是为什么需要等待这个时候才构建呢？笔者认为有两个原因：一个是内存占用原因，如果没有改proto文件没有被使用到，就不需要前置构建，占用内存；另一个是启动效率原因，没有必要为了没有被使用到的proto文件做无用功，而且就算后续使用到了再构建，也只是第一个使用者会牺牲一些效率。


最终会使用 DescriptorBuilder来进行 symbol 索引的构建：TryFindSymbolInFallbackDatabase() -> BuildFileFromDatabase() -> DescriptorBuilder().BuildFile(proto) -> BuildFileImpl()

```
bool DescriptorPool::TryFindSymbolInFallbackDatabase(
    absl::string_view name) const {
  if (fallback_database_ == nullptr) return false;

  if (tables_->known_bad_symbols_.contains(name)) return false;

  std::string name_string(name);
  auto file_proto = absl::make_unique<FileDescriptorProto>();
  if (  // We skip looking in the fallback database if the name is a sub-symbol
        // of any descriptor that already exists in the descriptor pool (except
        // for package descriptors).  This is valid because all symbols except
        // for packages are defined in a single file, so if the symbol exists
        // then we should already have its definition.
        //
        // The other reason to do this is to support "overriding" type
        // definitions by merging two databases that define the same type. (Yes,
        // people do this.)  The main difficulty with making this work is that
        // FindFileContainingSymbol() is allowed to return both false positives
        // (e.g., SimpleDescriptorDatabase, UpgradedDescriptorDatabase) and
        // false negatives (e.g. ProtoFileParser, SourceTreeDescriptorDatabase).
        // When two such databases are merged, looking up a non-existent
        // sub-symbol of a type that already exists in the descriptor pool can
        // result in an attempt to load multiple definitions of the same type.
        // The check below avoids this.
      IsSubSymbolOfBuiltType(name)

      // Look up file containing this symbol in fallback database.
      || !fallback_database_->FindFileContainingSymbol(name_string,
                                                       file_proto.get())

      // Check if we've already built this file. If so, it apparently doesn't
      // contain the symbol we're looking for.  Some DescriptorDatabases
      // return false positives.
      || tables_->FindFile(file_proto->name()) != nullptr

      // Build the file.
      || BuildFileFromDatabase(*file_proto) == nullptr) {
    tables_->known_bad_symbols_.insert(std::move(name_string));
    return false;
  }

  return true;
}

const FileDescriptor* DescriptorPool::BuildFileFromDatabase(
    const FileDescriptorProto& proto) const {
  mutex_->AssertHeld();
  build_started_ = true;
  if (tables_->known_bad_files_.contains(proto.name())) {
    return nullptr;
  }
  const FileDescriptor* result =
      DescriptorBuilder::New(this, tables_.get(), default_error_collector_)
          ->BuildFile(proto);
  if (result == nullptr) {
    tables_->known_bad_files_.insert(proto.name());
  }
  return result;
}

const FileDescriptor* DescriptorBuilder::BuildFile(
    const FileDescriptorProto& original_proto) {

    ...

    FileDescriptor* result = BuildFileImpl(proto, *alloc);
}

FileDescriptor* DescriptorBuilder::BuildFileImpl(
    const FileDescriptorProto& proto, internal::FlatAllocator& alloc) {

}
```

这里对TryFindSymbolInFallbackDatabase()进行一下简单说明：该函数会执行如下语句
```
fallback_database_->FindFileContainingSymbol(name_string,file_proto.get())
```

这会导致从我们前面构建的DescriptorPool中的EncodedDescriptorDatabase去查询，如果查不到那证明根本没有该message类型，否则就就会填充`file_proto`信息以便我们在后面进行构建。


接下来我们继续看BuildFileImpl()的实现：

```
FileDescriptor* DescriptorBuilder::BuildFileImpl(
    const FileDescriptorProto& proto, internal::FlatAllocator& alloc) {
  ...
  BUILD_ARRAY(proto, result, message_type, BuildMessage, nullptr);
  BUILD_ARRAY(proto, result, enum_type, BuildEnum, nullptr);
  BUILD_ARRAY(proto, result, service, BuildService, nullptr);
  BUILD_ARRAY(proto, result, extension, BuildExtension, nullptr);
  ...
}
```

BuildFileImpl 会针对每个 message_type、enum_type、service、extension 构建索引，举一个 BuildMessage 的例子。

```
void DescriptorBuilder::BuildMessage(const DescriptorProto& proto,
                                     const Descriptor* parent,
                                     Descriptor* result,
                                     internal::FlatAllocator& alloc) {
  ...
  BUILD_ARRAY(proto, result, oneof_decl, BuildOneof, result);
  BUILD_ARRAY(proto, result, field, BuildField, result);
  BUILD_ARRAY(proto, result, nested_type, BuildMessage, result);
  BUILD_ARRAY(proto, result, enum_type, BuildEnum, result);
  BUILD_ARRAY(proto, result, extension_range, BuildExtensionRange, result);
  BUILD_ARRAY(proto, result, extension, BuildExtension, result);
  BUILD_ARRAY(proto, result, reserved_range, BuildReservedRange, result);
  ...
  AddSymbol(...);
)
```

再看 AddSymbol 函数，从其代码可以看出会写两个表：一个是 symbols_by_name_ ，另一个是 symbols_by_parernt_ ，前者是通过命名来查找，后者是通过父类型调用触发的查找。

```
bool DescriptorBuilder::AddSymbol(const std::string& full_name,
                                  const void* parent, const std::string& name,
                                  const Message& proto, Symbol symbol) {
  ...
  if (tables_->AddSymbol(full_name, symbol)) {
    if (!file_tables_->AddAliasUnderParent(parent, name, symbol)) {
  ..
}
```

这两个表分布在不同的地方，symbols_by_name_ 是 DescriptorPool::Tables 类中，一般是全局搜索某个类型需要调用到，而symbols_by_parent_ 是在 FileDescriptorTables 类中，一般我们用来查询当前类型的某个字段（即field）用到比较多。


## 4. MessageFactory 索引

和DescriptorPool索引的构建时机相同，程序启动的时候构建了一部分索引，而在使用（也就是查询的时候）还会触发构建完整的Message索引数据。

### 4.1 MessageFactory索引的构建

我们还是接着前面AddDescriptorsImpl()接口开始，这个函数是在程序启动的时候触发执行的：
```
void AddDescriptorsImpl(const DescriptorTable* table) {
  // Reflection refers to the default fields so make sure they are initialized.
  internal::InitProtobufDefaults();
  internal::InitializeFileDescriptorDefaultInstances();

  // Ensure all dependent descriptors are registered to the generated descriptor
  // pool and message factory.
  int num_deps = table->num_deps;
  for (int i = 0; i < num_deps; i++) {
    // In case of weak fields deps[i] could be null.
    if (table->deps[i]) AddDescriptors(table->deps[i]);
  }

  // Register the descriptor of this file.
  DescriptorPool::InternalAddGeneratedFile(table->descriptor, table->size);
  MessageFactory::InternalRegisterGeneratedFile(table);
}

void MessageFactory::InternalRegisterGeneratedFile(
    const google::protobuf::internal::DescriptorTable* table) {
  GeneratedMessageFactory::singleton()->RegisterFile(table);
}

void GeneratedMessageFactory::RegisterFile(
    const google::protobuf::internal::DescriptorTable* table) {
  if (!files_.insert(table).second) {
    ABSL_LOG(FATAL) << "File is already registered: " << table->filename;
  }
}
```
GeneratedMessageFactory 类的定义如下(src/google/protobuf/message.cc)，我们主要关注两个成员`type_map_`和`files_` ，但实际上最有用的是`type_map_` ，`files_`只是辅助作用。那为什么这里只是构建了 type_map_ 呢？笔者认为和 DescriptorPool 索引的原因是一样的，一个是内存占用原因，另一个是启动效率原因。

```
class GeneratedMessageFactory final : public MessageFactory {
  void RegisterFile(const google::protobuf::internal::DescriptorTable* table);
  void RegisterType(const Descriptor* descriptor, const Message* prototype);

  // implements MessageFactory ---------------------------------------
  const Message* GetPrototype(const Descriptor* type) override;

  // Only written at static init time, so does not require locking.
  absl::flat_hash_set<const google::protobuf::internal::DescriptorTable*,
                      DescriptorByNameHash, DescriptorByNameEq>
      files_;

  absl::flat_hash_map<const Descriptor*, const Message*> type_map_
      ABSL_GUARDED_BY(mutex_);
};
```
### 4.2 MessageFactory索引的查询过程

从开发者怎么使用说起吧。开发者一般是调用 GetPrototype 函数来获取Messgae 实例:

```
const google::protobuf::Message* prototype
    = google::protobuf::MessageFactory::generated_factory()
      ->GetPrototype(descriptor);
```

这里我们来看看MessageFactory::GetPrototype()的实现(src/google/protobuf/message.cc)：

```
const Message* GeneratedMessageFactory::GetPrototype(const Descriptor* type) {
  {
    absl::ReaderMutexLock lock(&mutex_);
    const Message* result = FindInTypeMap(type);
    if (result != nullptr) return result;
  }

  // If the type is not in the generated pool, then we can't possibly handle
  // it.
  if (type->file()->pool() != DescriptorPool::generated_pool()) return nullptr;

  // Apparently the file hasn't been registered yet.  Let's do that now.
  const internal::DescriptorTable* registration_data =
      FindInFileMap(type->file()->name());
  if (registration_data == nullptr) {
    ABSL_DLOG(FATAL) << "File appears to be in generated pool but wasn't "
                        "registered: "
                     << type->file()->name();
    return nullptr;
  }

  absl::WriterMutexLock lock(&mutex_);

  // Check if another thread preempted us.
  const Message* result = FindInTypeMap(type);
  if (result == nullptr) {
    // Nope.  OK, register everything.
    internal::RegisterFileLevelMetadata(registration_data);
    // Should be here now.
    result = FindInTypeMap(type);
  }

  if (result == nullptr) {
    ABSL_DLOG(FATAL) << "Type appears to be in generated pool but wasn't "
                     << "registered: " << type->full_name();
  }

  return result;
}

const Message* FindInTypeMap(const Descriptor* type)
      ABSL_SHARED_LOCKS_REQUIRED(mutex_)
  {
    auto it = type_map_.find(type);
    if (it == type_map_.end()) return nullptr;
    return it->second;
  }
```
从上面我们看到，在GetPrototype()中首先尝试调用FindInTypeMap()来进行查找，如果查找不到则调用internal::RegisterFileLevelMetadata()来对proto级别的文件进行注册。下面我们来看一下这个注册过程：

```
void RegisterFileLevelMetadata(const DescriptorTable* table) {
  AssignDescriptors(table);
  RegisterAllTypesInternal(table->file_level_metadata, table->num_messages);
}
```

这里又分成两步：

- AssignDescriptors(): 为DescriptorTable中的file_level_metadata,file_level_enum_descriptors,file_level_service_descriptors指定descriptor

- RegisterAllTypesInternal(): 注册所有类型的实力(instance)

1） **AssignDescriptors()为DescriptorTable中相关字段指定descriptor**

```
void AssignDescriptors(const DescriptorTable* table) {
  MaybeInitializeLazyDescriptors(table);
  absl::call_once(*table->once, AssignDescriptorsImpl, table, table->is_eager);
}

void AssignDescriptorsImpl(const DescriptorTable* table, bool eager) {
  // Ensure the file descriptor is added to the pool.
  {
    // This only happens once per proto file. So a global mutex to serialize
    // calls to AddDescriptors.
    static absl::Mutex mu{absl::kConstInit};
    mu.Lock();
    AddDescriptors(table);
    mu.Unlock();
  }
  if (eager) {
    // Normally we do not want to eagerly build descriptors of our deps.
    // However if this proto is optimized for code size (ie using reflection)
    // and it has a message extending a custom option of a descriptor with that
    // message being optimized for code size as well. Building the descriptors
    // in this file requires parsing the serialized file descriptor, which now
    // requires parsing the message extension, which potentially requires
    // building the descriptor of the message extending one of the options.
    // However we are already updating descriptor pool under a lock. To prevent
    // this the compiler statically looks for this case and we just make sure we
    // first build the descriptors of all our dependencies, preventing the
    // deadlock.
    int num_deps = table->num_deps;
    for (int i = 0; i < num_deps; i++) {
      // In case of weak fields deps[i] could be null.
      if (table->deps[i]) {
        absl::call_once(*table->deps[i]->once, AssignDescriptorsImpl,
                        table->deps[i],
                        /*eager=*/true);
      }
    }
  }

  // Fill the arrays with pointers to descriptors and reflection classes.
  const FileDescriptor* file =
      DescriptorPool::internal_generated_pool()->FindFileByName(
          table->filename);
  ABSL_CHECK(file != nullptr);

  MessageFactory* factory = MessageFactory::generated_factory();

  AssignDescriptorsHelper helper(
      factory, table->file_level_metadata, table->file_level_enum_descriptors,
      table->schemas, table->default_instances, table->offsets);

  for (int i = 0; i < file->message_type_count(); i++) {
    helper.AssignMessageDescriptor(file->message_type(i));
  }

  for (int i = 0; i < file->enum_type_count(); i++) {
    helper.AssignEnumDescriptor(file->enum_type(i));
  }
  if (file->options().cc_generic_services()) {
    for (int i = 0; i < file->service_count(); i++) {
      table->file_level_service_descriptors[i] = file->service(i);
    }
  }
  MetadataOwner::Instance()->AddArray(table->file_level_metadata,
                                      helper.GetCurrentMetadataPtr());
}
```
我们在前面讲解使用DescriptorPool来构建索引的过程，通过DescriptorTable所构建出来的Descriptors只是加到了DescriptorPool中了。这里是从中取出各种Descriptor填充到DescriptorTable中的末尾三个字段中：

```
// This struct tries to reduce unnecessary padding.
// The num_xxx might not be close to their respective pointer, but this saves
// padding.
struct PROTOBUF_EXPORT DescriptorTable {
  mutable bool is_initialized;
  bool is_eager;
  int size;  // of serialized descriptor
  const char* descriptor;
  const char* filename;
  absl::once_flag* once;
  const DescriptorTable* const* deps;
  int num_deps;
  int num_messages;
  const MigrationSchema* schemas;
  const Message* const* default_instances;
  const uint32_t* offsets;
  // update the following descriptor arrays.
  Metadata* file_level_metadata;
  const EnumDescriptor** file_level_enum_descriptors;
  const ServiceDescriptor** file_level_service_descriptors;
};
```

2) **RegisterAllTypesInternal()注册所有类型的实例**

C++从语言层面本身不支持反射，如何从message type创建出真正的实例对象呢？ RegisterAllTypesInternal()的做法就是为每一个Type Descriptor保存一个默认的instance:

```
// Separate function because it needs to be a friend of
// Reflection
void RegisterAllTypesInternal(const Metadata* file_level_metadata, int size) {
  for (int i = 0; i < size; i++) {
    const Reflection* reflection = file_level_metadata[i].reflection;
    MessageFactory::InternalRegisterGeneratedMessage(
        file_level_metadata[i].descriptor,
        reflection->schema_.default_instance_);
  }
}

void MessageFactory::InternalRegisterGeneratedMessage(
    const Descriptor* descriptor, const Message* prototype) {
  GeneratedMessageFactory::singleton()->RegisterType(descriptor, prototype);
}

void GeneratedMessageFactory::RegisterType(const Descriptor* descriptor,
                                           const Message* prototype) {
  ABSL_DCHECK_EQ(descriptor->file()->pool(), DescriptorPool::generated_pool())
      << "Tried to register a non-generated type with the generated "
         "type registry.";

  // This should only be called as a result of calling a file registration
  // function during GetPrototype(), in which case we already have locked
  // the mutex.
  mutex_.AssertHeld();
  if (!type_map_.try_emplace(descriptor, prototype).second) {
    ABSL_DLOG(FATAL) << "Type is already registered: "
                     << descriptor->full_name();
  }
}
```

最终是把 prototype（也就是 reflection->schema.default_instance ）插入 type_map_ 表中。回到我们的echo_c++示例中，即是把如下两个instance插入到了`type_map_`表中：
```
PROTOBUF_ATTRIBUTE_NO_DESTROY PROTOBUF_CONSTINIT
    PROTOBUF_ATTRIBUTE_INIT_PRIORITY1 EchoRequestDefaultTypeInternal _EchoRequest_default_instance_;

PROTOBUF_ATTRIBUTE_NO_DESTROY PROTOBUF_CONSTINIT
    PROTOBUF_ATTRIBUTE_INIT_PRIORITY1 EchoResponseDefaultTypeInternal _EchoResponse_default_instance_;
```

因为Message 都实现了 New 函数，可以通过 default_instance->New()创建出 Message 实例。在echo_c++示例中有如下：

```
EchoRequest* New(::google::protobuf::Arena* arena = nullptr) const final {
    return CreateMaybeMessage<EchoRequest>(arena);
}
EchoResponse* New(::google::protobuf::Arena* arena = nullptr) const final {
    return CreateMaybeMessage<EchoResponse>(arena);
}
```
### 4.3 通过MessageFactory()创建message实例
前面我们讲到可以通过MessageFactory的如下方法获得prototype:
```
const google::protobuf::Message* prototype
    = google::protobuf::MessageFactory::generated_factory()
      ->GetPrototype(descriptor);
```
我们继续使用该prototype的New()方法就可以创建出真正的message实例了：
```
google::protobuf::Message* req_msg = prototype->New();
```

## 5. Reflection实现Message字段的修改

上面我们讲解到可以通过MessageFactory创建出Message，但是所创建出来的message是一个 google::protobuf::Message 基类指针，我们还是无法操作具体的message实例成员。这就需要我们的Reflection出场了。

通过Message的GetMetaData()方法我们就能获取到实际的message实例的MetaData:
```
  // Get a struct containing the metadata for the Message, which is used in turn
  // to implement GetDescriptor() and GetReflection() above.
  virtual Metadata GetMetadata() const = 0;
```
其中MetaData包含两个部分：Descriptor以及Reflection

```
// A container to hold message metadata.
struct Metadata {
  const Descriptor* descriptor;
  const Reflection* reflection;
};
```
Reflection类似一个代理人的角色，可以帮忙做一些读写的操作。比如像下面的 SetString()、GetString()函数(src/google/protobuf/message.h)：

```
class PROTOBUF_EXPORT Reflection final {
    std::string GetString(const Message& message,
                        const FieldDescriptor* field) const;
    void SetString(Message* message, const FieldDescriptor* field,
                        std::string value) const;
};
```
接着我们就来看看如何通过Refection来获取和设置message实例相关字段的值。不过在此之前，我们先介绍FieldDescriptor。

### 5.1 字段索引(FieldDescriptor)

我们在前面的「DescriptorPool索引的查询过程」一节中介绍了构建symbol 索引的过程，对于FieldDescriptor也是在那个时候构建的:
```
void DescriptorBuilder::BuildMessage(const DescriptorProto& proto,
                                     const Descriptor* parent,
                                     Descriptor* result,
                                     internal::FlatAllocator& alloc) {
  ...
  BUILD_ARRAY(proto, result, oneof_decl, BuildOneof, result);
  BUILD_ARRAY(proto, result, field, BuildField, result);
  BUILD_ARRAY(proto, result, nested_type, BuildMessage, result);
  BUILD_ARRAY(proto, result, enum_type, BuildEnum, result);
  BUILD_ARRAY(proto, result, extension_range, BuildExtensionRange, result);
  BUILD_ARRAY(proto, result, extension, BuildExtension, result);
  BUILD_ARRAY(proto, result, reserved_range, BuildReservedRange, result);
  ...
  AddSymbol(...);
)

void BuildField(const FieldDescriptorProto& proto, Descriptor* parent,
                  FieldDescriptor* result, internal::FlatAllocator& alloc) {
    BuildFieldOrExtension(proto, parent, result, false, alloc);
}

void DescriptorBuilder::BuildFieldOrExtension(const FieldDescriptorProto& proto,
                                              Descriptor* parent,
                                              FieldDescriptor* result,
                                              bool is_extension,
                                              internal::FlatAllocator& alloc) {
    AddSymbol(result->full_name(), parent, result->name(), proto, Symbol(result));
}

```
