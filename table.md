
🧱 Étape 1 — Tables Master / Référentiels (socle du système)

School

Region

Prefecture

SousPrefecture

Quartier

AcademicYearType

TermType

Term

Category (Cycle : Primaire, Collège, Lycée…)

Level (Niveau : 6e, 5e, 2nde…)

Track (Option/Série : Scientifique, Littéraire…)

Subject

SubjectChoice



---

👤 Étape 2 — Utilisateurs & Rôles

CustomUser

TeacherProfile

StudentProfile

ParentProfile

AdminProfile



---

🏫 Étape 3 — Structure scolaire (cœur du système)

SchoolYear

SchoolYearCategory

SchoolYearCategoryTerm  ← (règle clé que vous avez définie)

Classroom

ClassroomSubject

Enrollment



---

📝 Étape 4 — Évaluations & Notes (partie la plus contrainte)

EvaluationType (Academic / Internal)

EvaluationCategory (Devoir, Composition, Examen…)

Evaluation

EvaluationSubject

GradeType (type de note possible par matière)

Grade



---

📅 Étape 5 — Emploi du temps

TimeSlot (première heure, deuxième heure…)

Timetable

TimetableEntry



---

🧩 Tables transversales / contraintes métier importantes

AuditModel (abstraite, héritée par toutes les tables)

ClassroomSubjectCoefficient (si séparé pour performance)

StudentIdentifier (QR / numéro d’inscription unique)



---

🗂️ Vue condensée (toutes les tables)

School
Region / Prefecture / SousPrefecture / Quartier

AcademicYearType / TermType / Term
Category / Level / Track
Subject / SubjectChoice

CustomUser
TeacherProfile / StudentProfile / ParentProfile / AdminProfile

SchoolYear
SchoolYearCategory
SchoolYearCategoryTerm

Classroom
ClassroomSubject
Enrollment

EvaluationType
EvaluationCategory
Evaluation
EvaluationSubject
GradeType
Grade

TimeSlot
Timetable
TimetableEntry

StudentIdentifier
ClassroomSubjectCoefficient

