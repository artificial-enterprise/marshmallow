\
.. _dataclass_inference:

====================================
Schema Inference from Dataclasses
====================================

Marshmallow can automatically generate schema fields based on Python dataclasses.
This allows you to define your data structures once using dataclasses and then
derive Marshmallow schemas for serialization and validation with minimal boilerplate.

Basic Usage
-----------

There are two ways to use dataclass inference with Marshmallow:

1. Using the ``from_dataclass`` class method (recommended):

.. code-block:: python

    import dataclasses
    from marshmallow import Schema

    @dataclasses.dataclass
    class User:
        id: int
        name: str
        email: str | None = None # Equivalent to typing.Optional[str]
        is_active: bool = True
        tags: list[str] = dataclasses.field(default_factory=list)

    # Create a schema directly from the dataclass
    UserSchema = Schema.from_dataclass(User)
    
    # Load data into a User instance
    user_data = {"id": 1, "name": "John", "email": "john@example.com"}
    user = UserSchema().load(user_data)  # Returns a User instance

2. Using the ``dataclass`` attribute in the ``Meta`` class:

.. code-block:: python

    import dataclasses
    from marshmallow import Schema, fields

    @dataclasses.dataclass
    class User:
        id: int
        name: str
        email: str | None = None # Equivalent to typing.Optional[str]
        is_active: bool = True
        tags: list[str] = dataclasses.field(default_factory=list)

    class UserSchema(Schema):
        class Meta:
            dataclass = User

    # The UserSchema will now have fields:
    # id: fields.Integer(required=True)
    # name: fields.String(required=True)
    # email: fields.String(required=False, allow_none=True, load_default=None)
    # is_active: fields.Boolean(required=False, load_default=True)
    # tags: fields.List(fields.String(), required=False, load_default=list) # Note: current known issue with load_default
    
    # Note: With this approach, schema.load() returns a dict by default
    # You would need to add a post_load method to create a User instance

How Types are Mapped
--------------------

Marshmallow attempts to map Python types from dataclass field hints to appropriate
Marshmallow fields:

- ``int`` -> ``fields.Integer``
- ``float`` -> ``fields.Float``
- ``str`` -> ``fields.String``
- ``bool`` -> ``fields.Boolean``
- ``datetime.date`` -> ``fields.Date``
- ``datetime.datetime`` -> ``fields.DateTime``
- ``datetime.time`` -> ``fields.Time``
- ``datetime.timedelta`` -> ``fields.TimeDelta``
- ``decimal.Decimal`` -> ``fields.Decimal``
- ``uuid.UUID`` -> ``fields.UUID``
- ``typing.List[T]`` -> ``fields.List(marshmallow_field_for_T)``
- ``typing.Tuple[T, ...]`` -> ``fields.List(marshmallow_field_for_T)`` (for variable-length tuples)
- ``typing.Tuple[T1, T2]`` -> ``fields.Tuple([marshmallow_field_for_T1, marshmallow_field_for_T2])`` (for fixed-length tuples)
- ``typing.Dict[K, V]`` -> ``fields.Dict(keys=marshmallow_field_for_K, values=marshmallow_field_for_V)``
- ``typing.Union[T, None]`` (or ``typing.Optional[T]``) -> marshmallow field for T with ``required=False`` and ``allow_none=True``.
- Nested dataclasses: See :ref:`nested_dataclasses_inference`.

If a direct mapping is not found for a type, Marshmallow will raise an error during schema construction.
You can always override the inferred field or provide a custom field for complex types.

Handling of ``Optional``, Default Values, and ``default_factory``
-----------------------------------------------------------------

- **Required Fields**: Fields without a default value and not marked as ``Optional`` are inferred as ``required=True``.
- **Optional Fields (``type | None`` or ``typing.Optional[type]``)**:
    - Inferred as ``required=False`` and ``allow_none=True``.
    - If no default value is provided in the dataclass, ``load_default`` will be ``None``.
- **Fields with Default Values (e.g., ``name: str = "Guest"``)**:
    - Inferred as ``required=False``.
    - The ``load_default`` attribute of the field will be set to the dataclass field's default value.
- **Fields with ``default_factory`` (e.g., ``items: list = dataclasses.field(default_factory=list)``)**:
    - Inferred as ``required=False``.
    - The ``load_default`` attribute *should* be set to the factory callable.
    - **Known Issue**: Currently, ``load_default`` for fields with ``default_factory`` is incorrectly set to ``marshmallow.missing`` instead of the factory itself. This means that if the field is absent during deserialization, the default factory will not be invoked. This is a bug planned to be fixed.

Field Overriding and Adding Extra Fields
----------------------------------------

You can customize the inferred schema by explicitly defining fields in your schema class.
Explicitly defined fields will always take precedence over inferred fields.

.. code-block:: python

    @dataclasses.dataclass
    class Book:
        title: str
        author_email: str

    class BookSchema(Schema):
        # Override 'author_email' to use fields.Email for validation
        author_email = fields.Email(required=True)

        # Add an extra field not present in the dataclass
        publication_year = fields.Integer()

        class Meta:
            dataclass = Book

    # BookSchema fields:
    # title: fields.String(required=True) (inferred)
    # author_email: fields.Email(required=True) (explicitly defined)
    # publication_year: fields.Integer() (explicitly defined)

.. _nested_dataclasses_inference:

Nested Dataclasses and Schema Resolution
----------------------------------------

If a dataclass field is another dataclass, Marshmallow will attempt to infer a nested schema.

**Automatic Resolution by Name**:

By default, Marshmallow tries to find a schema for the nested dataclass by looking for a schema
class with the same name as the nested dataclass, appended with "Schema".

.. code-block:: python

    @dataclasses.dataclass
    class Author:
        name: str
        email: str

    @dataclasses.dataclass
    class Book:
        title: str
        author: Author # Nested dataclass

    # Schema for the nested dataclass
    class AuthorSchema(Schema):
        class Meta:
            dataclass = Author

    # Schema for the parent dataclass
    class BookSchema(Schema):
        class Meta:
            dataclass = Book

    # BookSchema will infer 'author' as fields.Nested(AuthorSchema)

**Explicit Schema Name**:

If the schema for the nested dataclass has a different name, or if you need to pass arguments
to the nested schema, you can explicitly define it:

.. code-block:: python

    @dataclasses.dataclass
    class AuthorDetails: # Renamed dataclass
        name: str
        email: str

    @dataclasses.dataclass
    class Publication: # Renamed dataclass
        title: str
        author: AuthorDetails

    class AuthorDetailsSchema(Schema): # Schema name matches new dataclass name
        class Meta:
            dataclass = AuthorDetails

    class PublicationSchema(Schema):
        # Explicitly define the nested field if auto-resolution might fail
        # or if you need to pass arguments (e.g., only, exclude)
        author = fields.Nested(AuthorDetailsSchema)

        class Meta:
            dataclass = Publication

You can also provide the name of the schema class as a string if it's defined later or to avoid
circular imports, similar to standard ``fields.Nested`` usage.

.. code-block:: python

    class ArticleSchema(Schema):
        author = fields.Nested("CreatorSchema") # Using string name

        class Meta:
            dataclass = Article # Assuming Article has an 'author: Creator' field

    @dataclasses.dataclass
    class Creator:
        name: str

    class CreatorSchema(Schema):
        class Meta:
            dataclass = Creator


Customizing Schema Name Resolution
----------------------------------

You can provide a custom function to resolve schema names for nested dataclasses
via ``Meta.schema_name_resolver``. This function takes the nested dataclass type
as an argument and should return the corresponding schema class or its string name.

.. code-block:: python

    def custom_resolver(dataclass_type):
        return f"{dataclass_type.__name__}GeneratedSchema"

    @dataclasses.dataclass
    class Profile:
        user_id: int

    # Assume ProfileGeneratedSchema is defined elsewhere or will be
    # class ProfileGeneratedSchema(Schema): ...

    @dataclasses.dataclass
    class Account:
        username: str
        profile: Profile

    class AccountSchema(Schema):
        class Meta:
            dataclass = Account
            schema_name_resolver = custom_resolver

    # AccountSchema will try to use 'ProfileGeneratedSchema' for the 'profile' field.

Loading Data
------------

Currently, when you use ``Schema.load(data)`` with a schema derived from a dataclass,
it will return a dictionary of deserialized data, not an instance of the dataclass.

.. code-block:: python

    user_data = {"id": 1, "name": "John Doe", "email": "john@example.com"}
    schema = UserSchema()
    loaded_data = schema.load(user_data)
    # loaded_data is {'id': 1, 'name': 'John Doe', 'email': 'john@example.com', 'is_active': True}
    # It is NOT an instance of User dataclass.

Automatic instantiation of the dataclass from the loaded data (e.g., via a ``@post_load``
method) is a planned future enhancement. You can, of course, add your own ``@post_load``
method to achieve this:

.. code-block:: python

    from marshmallow import post_load

    class UserSchemaWithLoad(Schema):
        class Meta:
            dataclass = User

        @post_load
        def make_user(self, data, **kwargs):
            return User(**data)

    schema_with_load = UserSchemaWithLoad()
    loaded_user_instance = schema_with_load.load(user_data)
    # loaded_user_instance is now an instance of User

This provides a flexible way to leverage dataclasses for defining your data structures
while using Marshmallow for robust serialization and validation.

Using the ``from_dataclass`` Method
-----------------------------------

The ``Schema.from_dataclass`` method provides a convenient way to create schemas directly from dataclasses.
It automatically infers fields from the dataclass type hints and, by default, creates instances of the dataclass
when deserializing data.

.. code-block:: python

    Schema.from_dataclass(
        dataclass_type,
        *,
        name=None,
        schema_name_resolver=None,
        auto_instantiate=True
    )

Parameters:

- ``dataclass_type``: The dataclass to generate a schema from.
- ``name``: Optional name for the schema class. If not provided, it will use the dataclass name + "Schema".
- ``schema_name_resolver``: Optional function to resolve schema names for nested dataclasses.
- ``auto_instantiate``: If True (default), automatically instantiate the dataclass from loaded data.
  If False, the schema will return dictionaries when deserializing data.

Example with all parameters:

.. code-block:: python

    @dataclasses.dataclass
    class Address:
        street: str
        city: str
        
    @dataclasses.dataclass
    class Person:
        name: str
        address: Address
        
    # Custom schema name resolver
    def my_resolver(dc_type):
        return f"Custom{dc_type.__name__}Schema"
        
    # Create schema with custom options
    PersonSchema = Schema.from_dataclass(
        Person,
        name="PersonDataSchema",
        schema_name_resolver=my_resolver,
        auto_instantiate=True
    )
    
    # This will look for a schema named "CustomAddressSchema" for the nested address field
