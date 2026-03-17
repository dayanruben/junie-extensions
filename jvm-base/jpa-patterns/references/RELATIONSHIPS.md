# JPA Entity Relationships Reference

## OneToMany / ManyToOne

```java
// ✅ Bidirectional — Author owns the relationship
@Entity
public class Author {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Book> books = new ArrayList<>();

    // Always sync both sides
    public void addBook(Book book) {
        books.add(book);
        book.setAuthor(this);
    }
    public void removeBook(Book book) {
        books.remove(book);
        book.setAuthor(null);
    }
}

@Entity
public class Book {
    @ManyToOne(fetch = FetchType.LAZY)  // always LAZY on the many side
    @JoinColumn(name = "author_id")
    private Author author;
}
```

---

## ManyToMany

```java
// ✅ Use Set (not List) to avoid duplicate join rows
@Entity
public class Student {
    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();

    public void enroll(Course course) {
        courses.add(course);
        course.getStudents().add(this);
    }
    public void drop(Course course) {
        courses.remove(course);
        course.getStudents().remove(this);
    }
}

@Entity
public class Course {
    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();
}
```

**Tip:** For ManyToMany with extra columns (e.g. `enrolledAt`), use an explicit join entity with two `@ManyToOne` instead.

---

## equals() and hashCode()

```java
// ✅ Use business key (natural ID), not database ID
@Entity
public class Book {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NaturalId
    @Column(unique = true, nullable = false)
    private String isbn;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Book book)) return false;
        return isbn != null && isbn.equals(book.isbn);
    }

    @Override
    public int hashCode() {
        return Objects.hash(isbn);  // stable — doesn't change after persist
    }
}
```

**Why not use `id`?** Before `save()`, `id` is `null`, so a transient entity is never equal to a persisted one. This breaks `Set` behavior and `removeBook()` helpers.

---

## Cascade Types

```java
// ✅ CascadeType.ALL only from parent to dependent children
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
private List<OrderItem> items;

// ❌ NEVER CascadeType.ALL on @ManyToOne — can delete shared entities!
@ManyToOne(cascade = CascadeType.ALL)  // deleting a Book deletes the Author!
private Author author;
```

| Cascade | Use when |
|---|---|
| `PERSIST` | Child should be saved with parent |
| `MERGE` | Child updates should propagate |
| `REMOVE` | Child should be deleted with parent |
| `ALL` | Full parent-owns-child relationship |
| `orphanRemoval = true` | Remove child when removed from collection |

---

## Common Mistakes

### toString() with lazy fields

```java
// ❌ Triggers lazy load (or throws if outside transaction)
@Override
public String toString() {
    return "Author{id=" + id + ", books=" + books + "}";
}

// ✅ Only include simple fields
@Override
public String toString() {
    return "Author{id=" + id + ", name='" + name + "'}";
}
```

### Missing index on FK column

```java
// ❌ findByAuthor queries do full table scan
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "author_id")
private Author author;

// ✅ Add index via @Table (standard JPA — works with all Hibernate versions)
@Entity
@Table(indexes = @Index(name = "idx_book_author_id", columnList = "author_id"))
public class Book { ... }
```
