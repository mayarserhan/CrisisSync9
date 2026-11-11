# CrisisSync — Project Story (Local AI Context)

> **Source:** `סיפור_CrisisSync.pdf`, five pages. This is a study/AI context file, not a submission artifact. The English section is a faithful working summary of the source, not a replacement for the original Hebrew text reproduced below. When in doubt, rely on the original PDF. Do not invent requirements.

## English working reference: functional and architectural facts

### Mission and scope

A nationwide rescue organization is replacing old management systems with a distributed desktop system, **CrisisSync**, to manage emergency incidents involving personnel, equipment, budgets and time. Tasks form dependency graphs, and all relevant stations receive status updates while multiple users work concurrently without double booking or race conditions. Login requires a username and password.

### User roles and permissions

1. **System Administrator**: administer user accounts and permissions, launch initial-data loading, inspect the read-only incident history, and view/generate administrative reports.
2. **Logistics Officer**: manage the equipment/personnel resource repository; change equipment maintenance status (faulty/in maintenance or available), move equipment between stations, and update exceptional crew unavailability such as leave or illness.
3. **Dispatcher**: create and cancel incidents, construct task dependency graphs, assign resources manually or automatically, monitor all open incidents in real time, cancel tasks, and close incidents through an external settlement system.
4. **Field Commander**: view incidents and tasks **for which they are the appointed commander**, and report those tasks complete. A commander is **both a system user and a personnel resource**; assignment does not disable login, but concurrent assignment to a second task is forbidden.
5. **Field Crew Member**: report start of shift and readiness (available/unavailable), view their assigned task, and report equipment failure from the field in real time.

### Resources

Each individual resource is atomic and can be reserved for at most **one task at a time**.

- **Personnel**: ID, name, one closed-list role (medic, heavy-equipment operator, firefighter, hazardous-materials specialist, rescuer, field commander), current location, and status (available, reserved, unavailable).
- **Equipment**: ID, one closed-list type (ambulance, fire engine, bulldozer, decontamination tent, generator), current location, and status (available, reserved, faulty/in maintenance). Faulty equipment may not be reserved. The logistics officer restores repaired equipment to available. If equipment breaks during an active task, a crew member or commander reports the fault; its status immediately becomes faulty and the dispatcher must assign replacement equipment.

### Incidents, tasks and dependencies

The dispatcher creates an incident with name, one closed-list type (building collapse, fire, road accident, hazardous-material leak, natural disaster), and location; creation timestamp is automatic.

Each task has a name, estimated duration, prerequisite tasks, and a required **resource bundle containing exactly one field commander**. Example: debris clearance requires a commander, heavy-equipment operator, bulldozer and medic. Task dependencies form a **directed acyclic graph (DAG)**; a task may have multiple prerequisites. Any addition or change that introduces a cycle is rejected with a message. Prerequisites can be added/modified **only while that task is Blocked**. If no unfinished prerequisite remains it transitions to Ready. Dependencies are locked in Ready, Active and final states. Resources cannot be reserved until all prerequisites have completed.

Task states:

- **Blocked**: waiting for unfinished prerequisites; a task created without open prerequisites starts directly in Ready.
- **Ready**: every prerequisite completed; resource reservation allowed.
- **Active**: the dispatcher reserved the **entire** required resource bundle in one transaction. If any resource is unavailable, the whole attempt fails and the task remains Ready.
- **Completed** (final): appointed field commander reports task completion; its resources are released immediately to available, and the server advances any dependent task whose prerequisites have all completed.
- **Cancelled** (final): dispatcher cancels a nonfinal task and releases its resources. All nonfinal tasks depending on it directly or indirectly are also cancelled and release their resources **only after dispatcher confirmation** of a message displaying how many tasks will be cancelled.

### Concurrent assignment and automatic allocation

The relational **SQL database is the sole source of truth**. Reservation uses a SQL transaction locking the requested resource rows (e.g. `SELECT ... FOR UPDATE` or ORM pessimistic locking), checks all are available, and updates all statuses atomically. If any is unavailable, roll back **all** changes and explain the conflict to the losing dispatcher; refresh their display. Two dispatchers reserving the same bulldozer at the same time must not both succeed.

The dispatcher may request automatic bundle allocation based on a selectable criterion, for example geographic proximity to incident location; future criteria must be easy to add. When concurrent allocations select the same resource, one succeeds while the other automatically attempts the next best available alternative (**up to two alternative attempts**) without making the dispatcher retry and without blocking other users. Allocation fails if no viable alternative exists in the repository.

### Live incident status view

Each open incident has a real-time view updated immediately via **server push**, **not polling**. Display incident name/type/location/elapsed time, completion progress (completed tasks versus total), dependency graph, and for each task its clearly distinguishable status, commander, assigned resources, and elapsed versus expected execution time. Overdue tasks are highlighted. Dispatchers see all open incident views and can act there; field commanders see only incidents with a task they command and can report completion there.

### Incident cancellation, closure and financial settlement

The dispatcher may cancel any open incident; all nonfinal tasks become Cancelled and their resources are released. Record the incident as cancelled and save cancellation time automatically.

**Only a dispatcher** may close an incident, and only when **all tasks are final** (Completed or Cancelled). On close: calculate operating costs from personnel hours, equipment engine hours and consumables; send a billing/budget reimbursement request to the external **National Clearing and Settlement System** representing municipalities, insurers and the finance ministry; if approved with a transaction token, persist the approval and permanently close the incident, recording closure time; if rejected, **do not close the incident** and notify the dispatcher. A **mock/stub adapter** must simulate approval/token and rejection for testing.

### Audit history and monthly reports

Maintain an audit log for every task state transition, resource update, incident creation/closure/cancellation, and settlement, including precise timestamp and acting user. Administrators can view open/closed/cancelled incidents filtered by date range, type and status and inspect tasks, reservations, settlement approvals and the full **read-only** action timeline.

Produce administrator-visible reports automatically at each month end using system time and also on demand for any selected past month:
- **Resource Utilization**: percentage of time during the month that each resource was reserved, shown graphically and aggregated by equipment type and personnel role.
- **SLA / Timeliness**: average interval from incident opening to completion of its **last task**, broken down by incident type.
- New similar reports should require minimal implementation changes.

### UI, architecture, implementation notes and academic constraints

Operation must be efficient and user-friendly, especially dispatcher/field commander screens under stress, while supporting safe future extension. This semester implements **stage 1 only**: a full multi-workstation **Java desktop** prototype in **IntelliJ IDEA** with a separate UI for each of five roles. Architecture: **JavaFX** desktop presentation strictly separate from business logic; **OCSF** over **TCP/IP LAN** for client/server communication; server-side business services for DAGs, state machines, concurrency, allocation; relational **SQL + ORM** for persistence and transactions. Stage 2 (future browser/mobile access over internet/cellular networks) is not required now but should be easy to add.

Use Git/GitHub versioning, Visual Paradigm formal UML (Use Case, Class, Sequence, Activity; Entity-Control-Boundary where appropriate), object-oriented separation of responsibilities, loose coupling, high cohesion, extensibility and course design patterns. Recommended Java distro: BellSoft Liberica Full JDK with JavaFX. Seed-data utility loads users/personnel/equipment from SQL script, JSON or CSV and is launchable by administrator but not part of normal operations. Demonstrate concurrent manual reservation and automatic alternative selection. Work is collaborative: all team members participate in all components. AI tools may be used for learning/design/testing, but submitted work and explanations are the team's responsibility; do not reuse other groups' work.

### Use Case model guardrails from Assignment 1 (separate assignment sheet)

- The formal model is in **Visual Paradigm 18.1**; PlantUML is recommended as an import source.
- Place **human actors to the left**, **external systems/scheduling actors to the right**, and show an explicit system boundary.
- Actor–use-case associations are solid **undirected** lines.
- `<<include>>` arrows point from the **base** use case to the included use case. `<<extend>>` arrows point from the **extension** to the extended/base use case.
- Include significant success/failure/exception paths. Add English specifications to the **main use cases** with preconditions, success/failure postconditions, numbered actor/system main flow, and alternatives tied to exact steps.
- In this phase, do **not** generate Assignment 1 Part B `REQUIREMENTS.md`.

---

## Original Hebrew text extracted from source PDF

> The text below is extracted from the supplied five-page PDF. Line breaks and some Hebrew/English punctuation or table-column alignment may differ from the PDF, so check the original PDF for authoritative layout.

### Source PDF — Page 1 of 5

מערכת CrisisSyncלטיפול בחירום

רקע
ארגון הצלה ארצי מחליף את מערכות הניהול הישנות שלו במערכת מבוזרת אחת ,CrisisSync ,לניהול אירועי
חירום הדורשים תיאום בין כוח אדם מקצועי ,ציוד הנדסי ורפואי ,תקציב וזמן.
המערכת מנהלת "עצי משימות" עם תלויות ( ,)Task Dependenciesומעדכנת שינויי סטטוס בכל התחנות בזמן
שמשתמשים רבים עובדים במקביל ,ללא התנגשויות נתונים (.)Race Conditions

משתמשים והרשאות
למערכת חמישה סוגי משתמשים .כל כניסה למערכת דורשת שם משתמש וסיסמה.

הרשאות משתמש
מנהל חשבונות משתמשים והרשאות ,מפעיל את מודול טעינת מנהל מערכת ( System
הנתונים (הערה ,)2צופה בהיסטוריית האירועים ומפיק דוחות )Administrator
ניהוליים.
מנהל את מאגר הציוד וכוח האדם :מעדכן סטטוס תחזוקה של ציוד קצין לוגיסטיקה ומשאבים
(תקול/בטיפול או זמין) ,מעביר כלים בין תחנות ,ומעדכן זמינות חריגה
של אנשי צוות (חופשה ,מחלה).
פותח ומבטל אירועים ,בונה עצי משימות ,משריין משאבים (ידנית או מוקדן ()Dispatcher
בהקצאה אוטומטית) ,עוקב אחר האירועים בזמן אמת ,וסוגר אירועים
מול מערכת ההתחשבנות החיצונית.
צופה באירועים ובמשימות שהוא מונה כמפקד שלהן ,ומדווח על סיומן. מפקד שטח ( Field
)Commander
מדווח על תחילת משמרת ומוכנות (זמין/לא זמין) ,צופה בפרטי איש צוות מבצעי ( Field Crew
המשימה שאליה שובץ ,ומדווח מהשטח על תקלה בציוד בזמן אמת. )Member

מאגר המשאבים
כל משאב (איש צוות או פריט ציוד) הוא ישות אטומית :בכל רגע הוא משוריין לכל היותר למשימה אחת.
כוח אדם .לכל איש צוות :מזהה ,שם ,תפקיד ,מיקום נוכחי וסטטוס (זמין ,משוריין למשימה ,לא זמין) .התפקיד
נבחר מרשימה סגורה :חובש ,מפעיל צמ"ה (ציוד מכני הנדסי) ,כבאי ,מומחה חומ"ס (חומרים מסוכנים) ,מחלץ,
מפקד שטח.
ציוד .הסוגים (רשימה סגורה) :אמבולנס ,כבאית ,דחפור ,אוהל טיהור ,גנרטור .לכל פריט :מזהה ,מיקום נוכחי
וסטטוס (זמין ,משוריין למשימה ,תקול/בטיפול) .פריט תקול אינו ניתן לשריון .קצין הלוגיסטיקה מעדכן סטטוס זה
ומחזיר את הפריט לזמין .חריג :אם כלי מתקלקל באמצע משימה פעילה ,איש הצוות או מפקד השטח מדווחים על
כך ,הפריט עובר מיד לסטטוס תקול ,והמוקדן נדרש להקצות פריט חלופי.
מעמד כפול של מפקד שטח .מפקד שטח הוא גם משתמש פעיל וגם משאב כוח אדם .כשהוא משוריין למשימה
הוא ממשיך לעבוד במערכת כרגיל ,אך המערכת חוסמת את שריונו למשימה נוספת במקביל .הוא רשאי לצפות
ולסגור רק משימות שהוא מונה כמפקד שלהן.

אירועים ועצי משימות
פתיחת אירוע .המוקדן פותח אירוע ומגדיר שם ,סוג (רשימה סגורה :קריסת מבנה ,שריפה ,תאונת דרכים,
דליפת חומ"ס ,אסון טבע) ומיקום .מועד הפתיחה נרשם אוטומטית.

---

### Source PDF — Page 2 of 5

משימות .לכל משימה המוקדן מגדיר :שם ,משך משוער ,משימות קדם וחבילת משאבים נדרשת .החבילה כוללת
תמיד מפקד שטח אחד בדיוק ,האחראי על המשימה .לדוגמה ,משימת "פינוי הריסות" דורשת מפקד שטח,
מפעיל צמ"ה ,דחפור וחובש.
תלויות .משימות האירוע יוצרות גרף מכוון חסר מעגלים ( :)DAGמשימה יכולה להיות תלויה בכמה משימות
קדם.
• בכל הוספה או שינוי של תלות המערכת בודקת מעגלים .פעולה שיוצרת מעגל נדחית ,והמוקדן מקבל הודעה
מתאימה.
אפשר להוסיף או לשנות משימות קדם רק למשימה במצב .Blockedאם לאחר השינוי לא נותרו לה משימות •
קדם פתוחות ,היא עוברת ל.Ready-
במצבים Active ,Readyובמצב סופי ,התלויות נעולות לשינוי. •
משימה אינה יכולה לשריין משאבים עד שכל משימות הקדם שלה הושלמו. •

מחזור החיים של משימה
בכל רגע משימה נמצאת באחד מחמשת המצבים:

משמעות מצב
ממתינה להשלמת משימות קדם .משימה שנוצרת עם משימת קדם פתוחה ( Blockedחסומה)
מתחילה במצב זה; משימה בלי תלויות פתוחות נוצרת ישירות במצב
.Ready
כל משימות הקדם הושלמו (השרת מעביר למצב זה אוטומטית) .אפשר ( Readyמוכנה)
לשריין לה משאבים.
המוקדן שריין את כל חבילת המשאבים בטרנזקציה אחת .אם משאב אחד ( Activeפעילה)
אינו זמין ,השריון כולו נכשל והמשימה נשארת במצב .Ready
מפקד השטח דיווח על סיום .המשאבים משתחררים מיד (חוזרים לזמין), ( Completedהושלמה,
והשרת מקדם את המשימות התלויות. סופי)
המוקדן ביטל את המשימה מכל מצב שאינו סופי ,ומשאביה שוחררו. ( Cancelledבוטלה ,סופי)
משימות התלויות בה (ישירות או בעקיפין) שטרם הגיעו למצב סופי
מבוטלות גם הן ומשאביהן משתחררים ,אחרי שהמוקדן מאשר הודעה
המציינת כמה משימות יבוטלו.

עבודה במקביל וסנכרון
מניעת שריון כפול ()Concurrency Control
אם שני מוקדנים מנסים לשריין באותו רגע את אותו דחפור (לאירוע בחיפה ולאירוע בתל אביב) ,רק אחד מהם
מצליח.
מסד הנתונים הרלציוני ( )SQLהוא מקור האמת הבלעדי. •
השריון מתבצע בטרנזקציית :SQLנעילת רשומות המשאבים המבוקשים (הערה ,)1בדיקה שכולם זמינים, •
ועדכון הסטטוס שלהם.
שריון חבילה הוא "הכול או כלום" :אם משאב אחד אינו זמין ,הטרנזקציה מבוטלת במלואה ( )Rollbackואף •
משאב אינו משוריין.
המוקדן שנכשל מקבל הודעה ברורה ,והמסך שלו מתעדכן למצב הנוכחי. •

---

### Source PDF — Page 3 of 5

הקצאה אוטומטית
בנוסף לשריון ידני ,המוקדן יכול לבקש הקצאה אוטומטית של החבילה לפי קריטריון נבחר (למשל ,המשאבים
הזמינים הקרובים ביותר גיאוגרפית למיקום האירוע) .יש לתכנן את המערכת כך שאפשר יהיה להוסיף
קריטריונים בעתיד.
אם שתי הקצאות לאירועים שונים בוחרות במקביל אותו משאב ,רק אחת מקבלת אותו .השנייה ממשיכה
אוטומטית לחלופה הזמינה הבאה בטיבה (עד 2ניסיונות חלופיים) ,בלי שהמוקדן יחזור על הבקשה ובלי לעכב
את השרת עבור משתמשים אחרים .ההקצאה נכשלת רק אם אין חלופה זמינה במאגר.

מסך סטטוס אירוע בזמן אמת
לכל אירוע פתוח יש מסך סטטוס חי .כל שינוי במצב משימה או משאב נדחף מהשרת מיד לכל התחנות
הרלוונטיות ( ,)Server Pushללא משיכה תקופתית ( )Pollingמצד הלקוח .המסך מציג:
פרטי האירוע :שם ,סוג ,מיקום וזמן שחלף מפתיחתו. •
מדד התקדמות :כמה משימות הושלמו מתוך הכלל. •
עץ המשימות והתלויות ביניהן. •
לכל משימה :מצב (בהבחנה ויזואלית ברורה בין המצבים) ,מפקד השטח ,המשאבים המשוריינים ,וזמן •
הביצוע שחלף לעומת המשך המשוער .משימה שחרגה מהמשך המשוער מסומנת בהדגשה.
המוקדן רואה את מסכי הסטטוס של כל האירועים הפתוחים ומבצע מתוכם את פעולותיו (שריון ,ביטול משימה,
סגירה וביטול של אירוע) .מפקד שטח רואה את מסכי האירועים שבהם הוא אחראי על משימה ,וסוגר מתוכם את
משימותיו.

סיום משימה ,סיום אירוע והתחשבנות
סגירת משימה .מפקד השטח מדווח על סיום והמשימה עוברת ל .Completed-השרת משחרר את משאביה
(חוזרים לזמין) ומעביר ל Ready-כל משימה תלויה שכל משימות הקדם שלה הושלמו.
ביטול אירוע .המוקדן יכול לבטל אירוע פתוח בכל שלב (למשל ,כשמתברר שמדובר בדיווח שווא) .כל המשימות
שטרם הגיעו למצב סופי עוברות ל Cancelled-ומשאביהן משתחררים .האירוע מסומן כמבוטל ,ומועד הביטול
נרשם אוטומטית.
סגירת אירוע והתחשבנות (הערה .)4רק המוקדן רשאי לסגור אירוע ,ורק אחרי שכל משימותיו במצב סופי
( Completedאו .)Cancelledתהליך הסגירה:
.1המערכת מחשבת את סך עלויות ההפעלה :שעות פעילות של כוח האדם ,שעות מנוע של הציוד ההנדסי
והרפואי ,וחומרים מתכלים.
.2המערכת שולחת דרישת חיוב ושיפוי תקציבי ל"מערכת סליקה והתחשבנות ארצית" ,גורם חיצוני המייצג את
הרשויות המקומיות ,חברות הביטוח ומשרד האוצר.
.3אם מתקבל אישור התחשבנות ומזהה עסקה ( ,)Transaction Tokenהאישור נרשם במסד הנתונים,
האירוע נסגר סופית ,ומועד הסגירה נרשם אוטומטית.
.4אם המערכת החיצונית דוחה את ההתחשבנות ,האירוע אינו נסגר והמוקדן מקבל התרעה מתאימה.

היסטוריית אירועים ()Audit Log
המערכת שומרת כל מעבר מצב של משימה ,כל עדכון משאב ,וכל פתיחה ,סגירה ,התחשבנות וביטול של אירוע,
יחד עם המועד המדויק והמשתמש שביצע את הפעולה.

---

### Source PDF — Page 4 of 5

מנהל המערכת יכול לצפות ברשימת האירועים (פתוחים ,סגורים ומבוטלים) ,לסנן אותה לפי טווח תאריכים ,סוג
אירוע ומצב ,ולהיכנס לאירוע כדי לראות את משימותיו ,המשאבים ששוריינו לכל משימה ,אישורי ההתחשבנות
וציר זמן מלא של הפעולות .ההיסטוריה לקריאה בלבד.

דוחות
בסוף כל חודש המערכת מפיקה אוטומטית דוחות ,שבהם צופה מנהל המערכת (הערה :)3
דו"ח ניצולת משאבים :אחוזי הניצול של הציוד ושל כוח האדם (החלק היחסי מזמן החודש שבו המשאב היה •
משוריין) ,בתצוגה גרפית ובחלוקה לסוגי ציוד ולתפקידים.
דו"ח עמידה בזמנים ( :)SLAהזמן הממוצע מפתיחת אירוע ועד סגירת המשימה האחרונה בו ,בחלוקה •
לסוגי אירועים.
צפויים בעתיד דוחות נוספים בעלי אופי דומה ,ולכן יש לבנות את המערכת כך שהוספת דו"ח חדש תדרוש עבודת
פיתוח מינימלית.

דרישות כלליות
התפעול צריך להיות יעיל וידידותי למשתמש ,במיוחד במסכי המוקדן ומפקד השטח ,המופעלים בתנאי לחץ .יש
לתכנן את המערכת לפי עקרונות התכנון שנלמדו בהרצאות ולפי הצרכים והאילוצים שלה :יעילה בתפעול ובעיבוד
הפנימי ,וגמישה לשינויים עתידיים באופן יעיל ובטוח.
המסמך כתוב בלשון זכר או נקבה לשם נוחות בלבד ,ומתייחס לשני המינים באופן שקול.

הגדרת הפרויקט וארכיטקטורה
עליכם לבנות ב Java-וב IntelliJ IDEA-מערכת תוכנה שמממשת את CrisisSyncלפי מסמך זה ,בהתאם
לשלבי תהליך פיתוח מערכות תוכנה ולהנחיות שיינתנו במהלך הסמסטר.
המערכת היא אפליקציה שולחנית (לא מבוססת )Webבארכיטקטורת שכבות מלאה (:)Full Stack

תיאור שכבה
ממשק משתמש עשיר ב ,JavaFX-בהפרדה מוחלטת מהלוגיקה העסקית. תצוגה ()Desktop GUI
תשתית Client-Serverברשת מקומית ( )LANמעל ,TCP/IPבאמצעות תקשורת שרת-לקוח
;OCSFמאפשרת תפעול בו-זמני מתחנות עבודה מרובות.
ניהול עצי המשימות ,אכיפת מכונות המצבים ,בקרת מקביליות ואלגוריתמי לוגיקה עסקית בשרת
הקצאה.
מסד נתונים רלציוני ( ,)SQLעם שכבת ORMלמיפוי ישויות וניהול גישה לנתונים
טרנזקציות. ()Persistence

התכנון יתבסס על עקרונות תכן מונחה-עצמים מתקדמים :הפרדת אחריות ,צימוד רפוי ,לכידות גבוהה ופתיחות
להרחבה .במטלות התכן תזהו נקודות במערכת הדורשות גמישות והרחבה ,ותשלבו בהן תבניות עיצוב ( Design
)Patternsשנלמדו בקורס.

שלבי הפיתוח
שלב ראשון :אב-טיפוס יישומי מלא שפועל מתחנות עבודה (מחשבים אישיים או ניידים) ,עם ממשק נפרד •
לכל אחד מחמשת סוגי המשתמשים .התקשורת לשרת ברשת מקומית ( )LANמעל TCP/IPו,OCSF-
בממשק שאינו מבוסס .Webהפרויקט הסמסטריאלי מקיף שלב זה.
שלב שני :גישה מכל מקום דרך האינטרנט (דפדפן) ודרך הרשת הסלולרית (אפליקציה ייעודית לכוחות •
השטח) .יש לתכנן את השלב הראשון כך שהמעבר לשלב השני יהיה יעיל וחלק ככל האפשר.

---

### Source PDF — Page 5 of 5

הערות למימוש
.1נעילת רשומות .אפשר בשאילתת SQLישירה (כגון )SELECT ... FOR UPDATEאו בנעילה פסימית
( )Pessimistic Lockingבשכבת ה .ORM-בהדגמה יש להראות ניסיון שריון בו-זמני של אותו משאב משתי
תחנות מוקדן ,שבו רק שריון אחד מצליח; וכן הקצאה אוטומטית בו-זמנית לשני אירועים המתחרים על אותו
משאב ,שבה אחד מהם עובר לחלופה.
.2טעינת נתונים .יש לממש מודול עזר ( )Utilityשטוען נתוני התחלה (משתמשים ,אנשי צוות ופריטי ציוד)
מקובץ נתונים (סקריפט JSON ,SQLאו )CSVאל מסד הנתונים .המודול אינו חלק מהתפעול השוטף ,אך
מנהל המערכת יכול להפעיל אותו בנוחות מתוך המערכת.
.3הפקת דוחות להדגמה .בנוסף להפקה האוטומטית בסוף החודש (לפי שעון המערכת) ,מנהל המערכת יכול
להפיק לפי דרישה דו"ח לחודש נבחר בעבר.
.4סימולציית מערכת הסליקה .יש לממש מתאם ( )Adapterהמדמה ( )Mock / Stubאת התקשורת עם
מערכת הסליקה וההתחשבנות .המתאם מקבל את סיכום העלויות ,מחזיר מזהה עסקה תקין ,ומאפשר גם
לדמות דחיית תשלום לבדיקת תרחישי קצה.

הנחיות כלליות
א .עבודה קבוצתית .כל מטלות הפרויקט הן עבודה קבוצתית .אין לחלק אותן בין חברי הקבוצה לעבודה
עצמאית; כל החברים משתתפים באופן פעיל בכל מרכיבי הפרויקט.
ב .שימוש חוזר ( .)Reuseשימוש חוזר פנימי וחיצוני הוא נושא מרכזי בקורס ונדרש בפרויקט .המרכיבים
לשימוש חוזר חיצוני ,תשתית OCSFוממשק ,JavaFXיוסברו ויתורגלו במהלך הסמסטר.
ג .סביבת פיתוח וניהול תצורה Java .ב( IntelliJ IDEA-מומלץ ,BellSoft Liberica Full JDKהכולל תמיכה
מובנית ב ;)JavaFX-שרת SQLרלציוני עם שכבת ;ORMניהול גרסאות ב Git-וב.GitHub-
ד .תכנון ומידול .המודל הפורמלי בדוחות הפרויקט ייבנה ב Visual Paradigm-לפי כללי הניתוח והתכן שנלמדו:
( Activity ,Sequence ,Class ,Use Caseבתצורת .)Entity-Control-Boundaryמומלץ להיעזר בכלי שרטוט
מבוססי קוד ( PlantUMLאו )Mermaidלגיבוש מהיר של המודלים.
ה .שימוש ב AI-ויושרה אקדמית .מותר ומעודד להשתמש בכלי בינה מלאכותית (מודלי שפה ,תוספי פיתוח כמו
Continueו )GitHub Copilot-ללמידה ,סיעור מוחות ארכיטקטוני ,כתיבת בדיקות ואיתור תקלות .התכנון והקוד
המוגשים הם באחריותם הבלעדית של חברי הקבוצה :כל חבר צוות נדרש להבין את הקוד והארכיטקטורה ,לנמק
את בחירות התכן ולבצע שינויים בזמן אמת במפגשי הבדיקה וההגנה .חל איסור מוחלט על שימוש במרכיבי
מטלות (מודל ,UMLקוד ,תשובות לשאלות וכל מרכיב אחר) מעבודות של סטודנטים או קבוצות אחרות,
בסמסטר זה או בסמסטרים קודמים.
תזכורת ,מתוך התקנון האקדמי של המכללה :הגשת בחינה ,עבודת בית ,עבודה סמינריונית ,עבודת גמר או עבודה אחרת
המוטלת על הסטודנט במסגרת לימודיו שאינה יצירה מקורית של הסטודנט ,ובכלל זה הצגת רעיונות של אחרים כשלו
והעתקת יצירות של אחרים ,גם אם בשינויים קלים והצגתם כשלו מהווים עבירה משמעתית.

שאלות .שאלות על פיתוח המערכת יש לפרסם בפורום המטלה הרלוונטי ב( Moodle-פורום מטלה ,1פורום
מטלה 2וכו') .הפורום הכללי מיועד לשאלות על החומר הנלמד בהרצאות ובתרגולים ולנושאים מנהלתיים.

בהצלחה!
