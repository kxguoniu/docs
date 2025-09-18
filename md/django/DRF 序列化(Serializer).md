[TOC]
## 摘要
序列化器允许将复杂的数据类型(如查询集、模型实例等)转化为原生`Python`数据类型，然后可以轻松将其呈现为`JSON`、`XML`或者其他数据类型。序列化器还提供反序列化，允许在验证数据之后将其转化为复杂数据类型。
## 基础序列化器
基础序列化器提供方法可以在创建实例时根据参数判断需要创建单一序列化器还是列表序列化器。除此之外还提供了保存入口、验证入口、序列化入口等方法。具体的实现由子类定义。
### 源码分析
```python
class BaseSerializer(Field):
    """
    In particular, if a `data=` argument is passed then:

    .is_valid() - Available.
    .initial_data - Available.
    .validated_data - Only available after calling `is_valid()`
    .errors - Only available after calling `is_valid()`
    .data - Only available after calling `is_valid()`

    If a `data=` argument is not passed then:
    .data - Available.
    """

    # 初始化类对象的方法
    def __init__(self, instance=None, data=empty, **kwargs):
        # 序列化的实例
        self.instance = instance
        # 反序列化的数据
        if data is not empty:
            self.initial_data = data
        # 局部更新设置
        self.partial = kwargs.pop('partial', False)
        # 上下文
        self._context = kwargs.pop('context', {})
        kwargs.pop('many', None)
        super().__init__(**kwargs)

    # 创建类对象，如果many=True，创建列表序列化类
    def __new__(cls, *args, **kwargs):
        if kwargs.pop('many', False):
            return cls.many_init(*args, **kwargs)
        return super().__new__(cls, *args, **kwargs)

    # 类方法，创建列表序列化对象
    @classmethod
    def many_init(cls, *args, **kwargs):
        allow_empty = kwargs.pop('allow_empty', None)
        child_serializer = cls(*args, **kwargs)
        list_kwargs = {
            'child': child_serializer,
        }
        if allow_empty is not None:
            list_kwargs['allow_empty'] = allow_empty
        list_kwargs.update({
            key: value for key, value in kwargs.items()
            if key in LIST_SERIALIZER_KWARGS
        })
        meta = getattr(cls, 'Meta', None)
        list_serializer_class = getattr(meta, 'list_serializer_class', ListSerializer)
        return list_serializer_class(*args, **list_kwargs)

    # 被子类重写的方法
    def to_internal_value(self, data):
        raise NotImplementedError('`to_internal_value()` must be implemented.')

    # instence -->  dict
    def to_representation(self, instance):
        raise NotImplementedError('`to_representation()` must be implemented.')

    # 更新数据库对象
    def update(self, instance, validated_data):
        raise NotImplementedError('`update()` must be implemented.')

    # 创建数据库对象
    def create(self, validated_data):
        raise NotImplementedError('`create()` must be implemented.')

    # 保存，根据参数选择是创建还是更新
    def save(self, **kwargs):
        assert not hasattr(self, 'save_object'), (
            'Serializer `%s.%s` has old-style version 2 `.save_object()` '
            'that is no longer compatible with REST framework 3. '
            'Use the new-style `.create()` and `.update()` methods instead.' %
            (self.__class__.__module__, self.__class__.__name__)
        )

        # 必须验证之后才可以保存数据
        assert hasattr(self, '_errors'), (
            'You must call `.is_valid()` before calling `.save()`.'
        )
        # 无效的数据
        assert not self.errors, (
            'You cannot call `.save()` on a serializer with invalid data.'
        )
		...

        validated_data = dict(
            list(self.validated_data.items()) +
            list(kwargs.items())
        )

        # 判断是更新还是创建
        if self.instance is not None:
            self.instance = self.update(self.instance, validated_data)
            assert self.instance is not None, (
                '`update()` did not return an object instance.'
            )
        else:
            self.instance = self.create(validated_data)
            assert self.instance is not None, (
                '`create()` did not return an object instance.'
            )

        return self.instance

    # 验证数据
    def is_valid(self, raise_exception=False):
        assert not hasattr(self, 'restore_object'), (
            'Serializer `%s.%s` has old-style version 2 `.restore_object()` '
            'that is no longer compatible with REST framework 3. '
            'Use the new-style `.create()` and `.update()` methods instead.' %
            (self.__class__.__module__, self.__class__.__name__)
        )

        assert hasattr(self, 'initial_data'), (
            'Cannot call `.is_valid()` as no `data=` keyword argument was '
            'passed when instantiating the serializer instance.'
        )

        # 如果数据没有被验证，则开始验证数据
        if not hasattr(self, '_validated_data'):
            try:
                self._validated_data = self.run_validation(self.initial_data)
            except ValidationError as exc:
                self._validated_data = {}
                self._errors = exc.detail
            else:
                self._errors = {}

        if self._errors and raise_exception:
            raise ValidationError(self.errors)

        return not bool(self._errors)

    # 返回序列化后的数据
    @property
    def data(self):
        if hasattr(self, 'initial_data') and not hasattr(self, '_validated_data'):
            msg = (
                'When a serializer is passed a `data` keyword argument you '
                'must call `.is_valid()` before attempting to access the '
                'serialized `.data` representation.\n'
                'You should either call `.is_valid()` first, '
                'or access `.initial_data` instead.'
            )
            raise AssertionError(msg)

        if not hasattr(self, '_data'):
            # 最简单的序列化
            if self.instance is not None and not getattr(self, '_errors', None):
                self._data = self.to_representation(self.instance)
            # 反序列化后的显示
            elif hasattr(self, '_validated_data') and not getattr(self, '_errors', None):
                self._data = self.to_representation(self.validated_data)
            else:
                self._data = self.get_initial()
        return self._data

    # 验证产生的异常
    @property
    def errors(self):
        if not hasattr(self, '_errors'):
            msg = 'You must call `.is_valid()` before accessing `.errors`.'
            raise AssertionError(msg)
        return self._errors

    # 经过验证的数据
    @property
    def validated_data(self):
        if not hasattr(self, '_validated_data'):
            msg = 'You must call `.is_valid()` before accessing `.validated_data`.'
            raise AssertionError(msg)
        return self._validated_data
```

## 单个序列化器
序列化是按照字段嵌套递归解析，反序列化亦是如此。  
验证数据的流程：

1. 检查数据合法性
2. 基础数据转化为复杂数据类型
   - 获取字段验证方法
   - 获取字段的值
   - 对字段的值进行递归验证，因为这个字段可能是一个嵌套字段
   - 使用字段验证方法处理上一步验证的结果
   - 设置字段的值
3. 运行验证器
4. 数据生效

### 源码分析
```python
class Serializer(BaseSerializer, metaclass=SerializerMetaclass):
    default_error_messages = {
        'invalid': _('Invalid data. Expected a dictionary, but got {datatype}.')
    }

    # 一个字段名称和字段实例的字典
    @cached_property
    def fields(self):
        fields = BindingDict(self)
        for key, value in self.get_fields().items():
            fields[key] = value
        return fields

    # 可写的字段实例生成器
    @property
    def _writable_fields(self):
        for field in self.fields.values():
            if not field.read_only:
                yield field

    # 可读的字段实例生成器
    @property
    def _readable_fields(self):
        for field in self.fields.values():
            if not field.write_only:
                yield field

    # 获取字段字典， 我们在子类中自定义设置的字段
    def get_fields(self):
        return copy.deepcopy(self._declared_fields)

    # Meta 属性中的验证器
    def get_validators(self):
        meta = getattr(self, 'Meta', None)
        validators = getattr(meta, 'validators', None)
        return list(validators) if validators else []

    # 获取初始数据字典，排除只读字段
    def get_initial(self):
        # 反序列化的初始数据，排除只读字段，排除没有赋值的字段
        if hasattr(self, 'initial_data'):
            # initial_data may not be a valid type
            if not isinstance(self.initial_data, Mapping):
                return OrderedDict()

            return OrderedDict([
                (field_name, field.get_value(self.initial_data))
                for field_name, field in self.fields.items()
                if (field.get_value(self.initial_data) is not empty) and
                not field.read_only
            ])

        # 序列化的初始数据，排除只读字段
        return OrderedDict([
            (field.field_name, field.get_initial())
            for field in self.fields.values()
            if not field.read_only
        ])

    # 获取字段的值
    def get_value(self, dictionary):
        # We override the default field access in order to support
        # nested HTML forms.
        if html.is_html_input(dictionary):
            return html.parse_html_dict(dictionary, prefix=self.field_name) or empty
        return dictionary.get(self.field_name, empty)

    # 检查、验证、生效
    def run_validation(self, data=empty):
        # 验证数据和字段是否匹配
        (is_empty_value, data) = self.validate_empty_values(data)
        if is_empty_value:
            return data

        # 原始类型 -> 字典数据类型
        value = self.to_internal_value(data)
        try:
            # 运行验证器
            self.run_validators(value)
            # 返回生效的数据
            value = self.validate(value)
            assert value is not None, '.validate() should return the validated data'
        except (ValidationError, DjangoValidationError) as exc:
            raise ValidationError(detail=as_serializer_error(exc))

        return value

    # 只读并且存在默认值
    def _read_only_defaults(self):
        fields = [
            field for field in self.fields.values()
            if (field.read_only) and (field.default != empty) and (field.source != '*') and ('.' not in field.source)
        ]

        defaults = OrderedDict()
        for field in fields:
            try:
                default = field.get_default()
            except SkipField:
                continue
            defaults[field.source] = default

        return defaults

    # 运行验证器之前将默认只读的字段添加进去
    def run_validators(self, value):
        if isinstance(value, dict):
            to_validate = self._read_only_defaults()
            to_validate.update(value)
        else:
            to_validate = value
        # 使用子类设置的验证列表对数据进行验证
        super().run_validators(to_validate)

    # 把需要验证的可写字段从源数据中提取出来，并使用字段验证器检查
    def to_internal_value(self, data):
        if not isinstance(data, Mapping):
            message = self.error_messages['invalid'].format(
                datatype=type(data).__name__
            )
            raise ValidationError({
                api_settings.NON_FIELD_ERRORS_KEY: [message]
            }, code='invalid')

        ret = OrderedDict()
        errors = OrderedDict()
        fields = self._writable_fields

        for field in fields:
            # 获取子类设置的字段验证方法
            validate_method = getattr(self, 'validate_' + field.field_name, None)
            # 返回字典中字段名称的值
            primitive_value = field.get_value(data)
            try:
                # 对字段进行检查、验证、生效。返回生效的数据
                validated_value = field.run_validation(primitive_value)
                # 使用设置的验证方法对生效的数据二次验证
                if validate_method is not None:
                    validated_value = validate_method(validated_value)
            except ValidationError as exc:
                errors[field.field_name] = exc.detail
            except DjangoValidationError as exc:
                errors[field.field_name] = get_error_detail(exc)
            except SkipField:
                pass
            else:
                # 设置字段的值
                set_value(ret, field.source_attrs, validated_value)

        if errors:
            raise ValidationError(errors)

        return ret

    # 框架数据类型装换成基础数据类型，只有可读字段
    def to_representation(self, instance):
        ret = OrderedDict()
        fields = self._readable_fields

        # 从实例中获取字段的值
        for field in fields:
            try:
                attribute = field.get_attribute(instance)
            except SkipField:
                continue

            # 默认外键使用pk优化
            check_for_none = attribute.pk if isinstance(attribute, PKOnlyObject) else attribute
            if check_for_none is None:
                ret[field.field_name] = None
            # 使用字段对象的方法解析内容，如果是序列化则进入序列化对象中解析(递归)
            else:
                ret[field.field_name] = field.to_representation(attribute)

        return ret

    # 是经过验证的数据生效
    def validate(self, attrs):
        return attrs

    def __repr__(self):
        return representation.serializer_repr(self, indent=1)

    # 每一个字段都包装成一个对象，可以向from一样的api

    def __iter__(self):
        for field in self.fields.values():
            yield self[field.field_name]

    def __getitem__(self, key):
        field = self.fields[key]
        value = self.data.get(key)
        error = self.errors.get(key) if hasattr(self, '_errors') else None
        if isinstance(field, Serializer):
            return NestedBoundField(field, value, error)
        if isinstance(field, JSONField):
            return JSONBoundField(field, value, error)
        return BoundField(field, value, error)

    # 返回字段的序列化信息
    @property
    def data(self):
        ret = super().data
        return ReturnDict(ret, serializer=self)

    # 返回验证的错误字典
    @property
    def errors(self):
        ret = super().errors
        if isinstance(ret, list) and len(ret) == 1 and getattr(ret[0], 'code', None) == 'null':
            # Edge case. Provide a more descriptive error than
            # "this field may not be null", when no data is passed.
            detail = ErrorDetail('No data provided', code='null')
            ret = {api_settings.NON_FIELD_ERRORS_KEY: [detail]}
        return ReturnDict(ret, serializer=self)
```
### 示例
```python
from rest_framework import serializers
class Example(serializers.Serializer):
    name = serial.CharField()
    age = serial.IntegerField()

    # 需要自己实现创建和更新操作
    def create(self, validated_data):
        pass
    def update(self, instance, validated_data):
        pass

    # 字段验证器
    def validate_name(self, name):
        if len(name) > 255:
            raise
        else:
            return name
    def validate_age(self, age):
        if age < 0:
            raise
        elif age > 150:
            raise
        else:
            return age

    # 全局验证
    def model_validators(self, value):
        assert "name" in value, "name must be"
        assert "age" in value, "age must be"
        return value

    class Meta:
        validators = [self.model_validators]
```

## 列表序列化器
`ListSerializer`类提供了序列化和一次验证多个对象的方法，您通常不需要直接使用`ListSerializer`。而是应该在实例化序列化器是传递`many=True`
### 示例
```python
from rest_framework import serializers
class BookListSerializer(serializers.ListSerializer):
    def update(self, instance, validated_data):
        book_mapping = {book.id: book for book in instance}
        data_mapping = {item["id"]: item for item in validated_data}

        ret = []
        for book_id, data in data_mapping.items():
            book = book_mapping.get(book_id, None)
            if book is None:
                ret.append(self.child.create(data))
            else:
                ret.append(self.child.update(book, data))

        for book_id, book in book_mapping.items():
            if book_id not in data_mapping:
                book.delete()

        return ret

class BookSerializer(serializers.Serializer):
    id = serializers.IntegerField()
    ...
    class Meta:
        list_serializer_class = BookListSerializer
```
## 模型序列化器
`ModelSerializer`继承自`Serializer`，它会根据模型自动生成一组字段、为序列化器生成默认的验证器、`.create()`and`.update()`的默认实现。
### 源码分析
```python
class ModelSerializer(Serializer):
    ...
    # 默认实现不处理嵌套关系，如若想要实现则可以自己重写
    def create(self, validated_data):
        raise_errors_on_nested_writes('create', self, validated_data)

        ModelClass = self.Meta.model

        # 从字段中删除多对多关系，应为它们要求是已经保存的对象
        info = model_meta.get_field_info(ModelClass)
        many_to_many = {}
        for field_name, relation_info in info.relations.items():
            if relation_info.to_many and (field_name in validated_data):
                many_to_many[field_name] = validated_data.pop(field_name)

        try:
            instance = ModelClass._default_manager.create(**validated_data)
        except TypeError:
            ...
            raise TypeError(msg)

        # 创建实例之后保存多对多关系
        if many_to_many:
            for field_name, value in many_to_many.items():
                field = getattr(instance, field_name)
                field.set(value)

        return instance

    def update(self, instance, validated_data):
        raise_errors_on_nested_writes('update', self, validated_data)
        info = model_meta.get_field_info(instance)

        m2m_fields = []
        for attr, value in validated_data.items():
            if attr in info.relations and info.relations[attr].to_many:
                m2m_fields.append((attr, value))
            else:
                setattr(instance, attr, value)

        instance.save()

        # 保存之后在设置对多对字段
        for attr, value in m2m_fields:
            field = getattr(instance, attr)
            field.set(value)

        return instance

    # 如果子类没有设置验证器则使用默认的验证器集
    def get_validators(self):
        # If the validators have been declared explicitly then use that.
        validators = getattr(getattr(self, 'Meta', None), 'validators', None)
        if validators is not None:
            return list(validators)

        # Otherwise use the default set of validators.
        return (
            self.get_unique_together_validators() +
            self.get_unique_for_date_validators()
        )
```
### 示例
```python
from rest_framework import serializers
class ExampleSerializer(serializers.ModelSerializer):
    # 根据参数选择序列化的字段
    def __init__(self, *args, **kwargs):
        allow_fields = kwargs.pop("fields", None)
        super().__init__(*args, **kwargs)
        if allow_fields is not None:
            allowd = set(allow_fields)
            existing = set(self.fields)
            for field_name in existing - allowd:
                self.fields.pop(field_name)

    # 字段参数
    # 只读、只写、必要字段、默认值、初始值、验证器列表、允许为空
    # read_only、write_only、required、default、initial、validators、allow_null

    # 列表序列化参数
    # 实例、数据、局部更新、上下文、许空
    # instance、data、partial、context、allow_empty
    group = GroupSerializer(many=False, read_only=False)
    class Meta:
        # 序列化模型
        model = User
        # 关系查找深度,默认0
        depth = 0
        # 序列化的字段
        fields = ["id", "name", "age"]
        # 指定排除的字段
        exclude = ["group"]
        # 指定只读字段
        read_only_fields = ["name", "age"]
        # 额外参数字段,如果已经显示声明，额外字段将被忽略
        extra_kwargs = {
            "name": {'read_only': True},
            "age": {'required': False}
        }
        # 验证器
        validators = []
        # 指定列表序列化器
        list_serializer_class = ListSerializer
```