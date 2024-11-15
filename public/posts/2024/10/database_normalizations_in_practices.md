# Database Normalizations In Practices

Database normalization is a process used to organize a database into tables and columns.

The goal is to reduce redundancy and dependency by dividing large tables into smaller, related ones.

This process moves through various "normal forms" (NF), each addressing specific types of redundancy.

In this blog, we’ll walk through 4nf normalization process aspectes: 
- Definitions: Conditions to make a table follow normalizations
- Practices: Handons on examples to make table sastify all normalizations.  
- Advantages (WIP): Benifits of design database in normalizations
- Drawback Acknowdelgements (WIP): Downsides of strictly following 4NF.

## 1NF: First Normal Form

### Definitions
#### 1NF
A relation is in **1NF** if:
- All values in a column are atomic.
- Columns don't contain arrays or compound values.
#### 2NF
A relation is in **2NF** if:
- It is in 1NF.
- There are no partial dependencies: non-key fields don't depend only on part of the composite keys.
#### 3NF
A relation is in **3NF** if:
- It is in 2NF.
- There are no transitive dependencies: non-key attributes don't depend on each others.
#### BCNF (Boyce-Codd Normal Form - a.k.a 3.5NF)
A relation is in **BCNF** if:
- It is in 3NF.
- The depended fields must be primary keys.
#### 4NF
In a dependency like A → B, if one value of A links to several values of B, this shows a multi-valued dependency in the relation.

A relation will be in 4NF if it is in Boyce Codd normal form and has no multi-valued dependency.

### Practices

Here’s a "classical" table that stores information about students, courses, and their preferred sports:

|StudentID(pk)| Courses(pk)     | Sports(pk)         | Name   |Instructor            |
|-------------|-----------------|--------------------|--------|----------------------|
| 1           | Math, Physics   | Football, Tennis   | Alice  |Dr. Smith, Dr. Johnson|
| 2           | Biology, Chem   | Basketball, Basebal| Bob    |Dr. Lee, Dr. Davis    |

This table isn't in 1NF.

We will restructure it step by step to make it sastify 4NF conditions.

#### Road to 1NF
In the original table, both the `Courses` and `Sports` columns contain multiple values, violating the First Normal Form (1NF).

To transform this table into 1NF, we must ensure each column holds only atomic (single) values.

| StudentID(pk) | Course   | Sport      | Name  | Instructor  |
|---------------|----------|------------|-------|-------------|
| 1             | Math     | Football   | Alice | Dr. Smith   |
| 1             | Math     | Tennis     | Alice | Dr. Smith   |
| 1             | Physics  | Football   | Alice | Dr. Johnson |
| 1             | Physics  | Tennis     | Alice | Dr. Johnson |
| 2             | Biology  | Basketball | Bob   | Dr. Lee     |
| 2             | Biology  | Baseball   | Bob   | Dr. Lee     |
| 2             | Chemistry| Basketball | Bob   | Dr. Davis   |
| 2             | Chemistry| Baseball   | Bob   | Dr. Davis   |

Each cell in the table contains only one value, which means the table is in the First Normal Form (1NF).

## Road to 2NF

The table from 1NF has a composite key (`StudentID`, `Course`, `Sport`), but `Name` depends only on `StudentID` and not on the full composite key.

It is a partial dependency and violates 2NF.

To remove the partial dependency, we split the table into two:
- A table for student details.
- A table for course and sport data.

**Student Table**

|StudentID(pk)| Name  |
|-------------|-------|
| 1           | Alice |
| 2           | Bob   |

**Student Course and Sport Table**

|StudentID(pk)| Course   | Sport     |Instructor|
|-------------|----------|-----------|----------|
| 1           | Math     | Football  |Dr. Smith |
| 1           | Physics  | Tennis    |Dr. Tennis|
| 2           | Biology  | Baseball  |Dr. Lee   |
| 2           | Chemistry| Baseball  |Dr. Davis |

Now, all non-key columns depend on the whole key, making both tables in 2NF.

---

## 3NF: Third Normal Form

In the last tables, `Instructor` depends on `Course`, not directly on the `StudentID`:

It is a transitive dependency.

To eliminate the transitive dependency, we need to separate the instructor information into another table:

#### Student Course and Sport Table (3NF)

|StudentID    | Course     | Sport     |
|-------------|------------|-----------|
| 1           | Math       | Football  |
| 1           | Physics    | Tennis    |
| 2           | Biology    | Baseball  |

#### Course Instructor Table (3NF)

|StudentID    | Course    | Instructor|
|-------------|-----------|-----------|
| 1           | Math      | Dr. Smith |
| 1           | Physics   | Dr. Brown |
| 2           | Biology   | Dr. Green |

Now, there are no transitive dependencies, so the tables are in 3NF.

---

## Road to BCNF
In the table Course Instructor Table, `Course` depends on `Instructor` (each course is taught by a single instructor), but `Course` is not a superkey because multiple students can enroll in the same course.
  
This violates BCNF.

To bring the table into BCNF, we need to split it into two tables, one for student-course information and another for course-instructor assignments:

#### Student Course Table (BCNF)

| StudentID | Course(FK)| Sport     |
|-----------|-----------|-----------|
| 1         | Math      | Football  |
| 1         | Physics   | Tennis    |
| 2         | Math      | Baseball  |

#### Course Instructor Table (BCNF)

| Course(PK)| Instructor|
|-----------|-----------|
| Math      | Dr. Smith |
| Physics   | Dr. Brown |
| Math      | Dr. Green |

Now, each functional dependency is properly represented, and the tables are in BCNF.

## Road to 4NF

Let’s look at a situation where a student can take several courses and play different sports.
 
In this case, the `Courses` and `Sports` are not related to each other. 

This creates multi-valued dependency problem, which goes against Fourth Normal Form (4NF).
 
To fix this and meet 4NF, we should split the course and sport information into two separate tables.

#### Student Course Table (4NF)

| StudentID | Course   |
|-----------|----------|
| 1         | Math     |
| 1         | Physics  |
| 2         | Biology  |
| 2         | Chemistry|

#### Student Sport Table (4NF)

| StudentID | Sport     |
|-----------|-----------|
| 1         | Football  |
| 1         | Basketball|
| 2         | Tennis    |
| 2         | Baseball  |

Now, each table contains only one independent fact about the student, and the design is in 4NF.

## Conclusion

Database normalization is an important part of creating efficient databases.

It involves several steps that remove different types of redundancy, dependency:
 
- **1NF** (First Normal Form) removes repeating groups.
- **2NF** (Second Normal Form) eliminates partial dependencies.
- **3NF** (Third Normal Form) removes transitive dependencies.
- **BCNF** (Boyce-Codd Normal Form) ensures all functional dependencies rely on superkeys.
- **4NF** (Fourth Normal Form) eliminates multi-valued dependencies.
