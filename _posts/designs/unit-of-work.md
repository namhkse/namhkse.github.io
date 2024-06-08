# Unit of Work Pattern

![Random image](https://pic.re/image)

## Intent
**Unit of Work Pattern** is...

The unit of work patterns are intended to create an abstraction layer between the data access layer and the business logic layer.
Implementing these patterns can help insulate your application from changes in the data store and can facilitate automated unit testing
or test-driven development (TDD).

- When the controller runs under a web server, it receives a repository that works with the Entity Framework.
- When the controller runs under a unit test class, it receives a repository that works with data stored in a way that you can easily manipulate for testing, such as an in-memory collection.

## Problem
You could instantiate a new context in the repository, but then if you used multiple 
repositories in one controller, each would end up with a **separate context**.

How a unit of work class can ensure that all repositories use the same context.

Suppose you have to update two different entity types as part of the same transaction.
If each uses a separate database context instance, **one might succeed and the other might fail**.

**No repository**
```mermaid
flowchart LR
  IIS --> Controller 
  Controller --> DbContext 
  DbContext --> Database
```

**With repository**
```mermaid
flowchart LR
  IIS --> Controller
  Controller --> Repostory1
  Controller --> Repostory2

  subgraph UnitOfWork 
    Repostory1 --> DbContext
    Repostory2 --> DbContext
  end

  DbContext --> Database
```
**With repository in testing**
```mermaid
flowchart LR
  IIS --> Controller
  Controller --> MockRepostory1
  Controller --> MockRepostory2

  subgraph UnitOfWork 
    MockRepostory1
    MockRepostory2
  end
```

## Solution
Implement a repository class for each entity type. Repository interface and class for Student entity

Use multiple repositories and a unit of work class for the Course and Department entity types in the Course controller. 

The unit of work class coordinates the work of multiple repositories by creating a single database context class shared by all of them. 

## Structure
Implement repository pattern:
```mermaid
classDiagram

StudentController --> IStudentRepository
StudentRepository ..|> IStudentRepository

class IStudentRepository {
  <<interface>>
  +GetStudents() IEnumerable<Student>
  +GetStudentByID(studentId: int)
  +InsertStudent(student: Student)
  +DeleteStudent(studentID: int)
  +UpdateStudent(student: Student)
  +Save()
}

class StudentRepository {
  -context: SchoolContext
  +GetStudents() IEnumerable<Student>
  +GetStudentByID(studentId: int)
  +InsertStudent(student: Student)
  +DeleteStudent(studentID: int)
  +UpdateStudent(student: Student)
  +Save()
}

class StudentController {
  -studentRepository: IStudentRepository
  + Get(...)
  + Create(student: Student)
}
```

Implement unit of work pattern:
```mermaid
classDiagram

CourseController --> UnitOfWork
UnitOfWork --> ICourseRepository
UnitOfWork --> IDepartmentRepository
CourseRepository ..|> ICourseRepository
CourseRepository --|> GenericRepository~Course~ 
DepartmentRepository ..|> IDepartmentRepository
DepartmentRepository --|> GenericRepository~Department~ 

class GenericRepository~TEntity~{
  SchoolContext context
  DbSet~TEntity~ dbSet

  +Get() IEnumerable~TEntity~
  +GetByID(object id) TEntity
  +Insert(TEntity entity)
  +Delete(object id)
  +Delete(TEntity entityToDelete)
  +Update(TEntity entityToUpdate)
}

class UnitOfWork {
  SchoolContext context;
  GenericRepository~Department~ departmentRepository
  GenericRepository~Course~ courseRepository

  +DepartmentRepository() GenericRepository~Department~
  +CourseRepository() GenericRepository~Course~
  +Save()
}
```

```csharp
class UnitOfWork {
  private SchoolContext context = new SchoolContext();
  private GenericRepository<Department> departmentRepository;
  private GenericRepository<Course> courseRepository;

  public GenericRepository<Department> DepartmentRepository()
  {
    if (this.departmentRepository == null)
    {
      this.departmentRepository = new GenericRepository<Department>(context);
    }
    return departmentRepository;
  }

  public GenericRepository<Course> CourseRepository() {
    // Same above
  }
}

class StudentController
{
  private IStudentRepository studentRepository;

  /// The default (parameterless) constructor creates a new context instance,
  public StudentController()
  {
    this.studentRepository = new StudentRepository(new SchoolContext());
  }

  /// Optional constructor allows the caller to pass in a context instance.
  public StudentController(IStudentRepository studentRepository)
  {
    this.studentRepository = studentRepository;
  }

  [HttpPost]
  public object Create(Student student)
  {
    studentRepository.InsertStudent(student);
    studentRepository.Save();
    return student;
  }
}
```
[Code example](https://refactoring.guru/design-patterns/abstract-factory)

## Applicability
1. Group multiple operations, that form a single unit of work.
2. Without the unit of work pattern, the repositories have to manage the transaction’s lifetime.
3. Unit of Work is an abstraction above repositories for transaction management

## Pros and Cons
**Pros**
- Data consistency, group operations together and commit or roll them back as a single unit.
- Performance, groups operations together and sends them through the wire in a batch, in
most cases it will be faster than committing transactions one by one. It is also unnecessary
to open and close multiple connections and manage new transactions one after another.

**Cons**
- Must evaluate before integrating a pattern into a specific project.
- Complexity, Introducing abstraction always increases code complexity too.
- Delayed State Changes, let’s take an example:
> Scenario where we want to create two related entities in the same transaction: an order and a payment. 
> Assume that we have database-generated IDs for both entities. In our code, when we create an order, it won’t possess an ID at that moment. 
> This is because the ID will be generated by the database, but the unit of work’s transaction has not been finalized as of now. 
>When creating a payment, we won’t be able to link it to the order, because it doesn’t have an ID yet.

 

## Relations with Other Patterns