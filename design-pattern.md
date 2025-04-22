#### Q1 you have and Order class with many optional fields. what pattern would you use to create immutable Order object efficiently?

**Answer**: 
1. Using *Builder Pattern*. with the builder pattern the object can be constructed incrementally and optional fields can be set without the need for complex constructors or multiple setter methods.

2. why to use buiilder pattern?

    - Immutatbility: The order object will be immutable meaning that once it is created its state cannot be changed.
    - Handling the optional fields: easily handles optional fields without clustering the constructor with many parameters. Each field is set only if necessary.

