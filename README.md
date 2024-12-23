# Постановка Завдання CSP

Завдання задоволення обмежень (CSP) базується на трьох основних елементах:

1. **Змінні**: Елементи, яким необхідно призначити значення.
2. **Домени**: Набори допустимих значень для кожної змінної.
3. **Обмеження**: Правила, що визначають прийнятні комбінації значень для змінних.

Для ефективного розв’язання задачі застосовуються різні підходи, такі як MRV (мінімальна кількість доступних значень), степенева евристика та LCV (значення, що найменше обмежує).

## 1. Змінні

### Представлення у Коді

Змінними є заняття (`Lesson`), які слід включити до розкладу. Їх структура визначена класом:

```python
class Lesson:
    def __init__(self, lesson_id, subject, lesson_type, group, subgroup=None):
        self.id = lesson_id
        self.subject = subject
        self.type = lesson_type
        self.group = group
        self.subgroup = subgroup
```

### Пояснення

## Компоненти:
- **Ідентифікатор:** Унікальний код заняття.
- **Предмет:** Дисципліна, що викладається.
- **Тип:** Категорія заняття (наприклад, лекція чи практика).
- **Група:** Студенти, які беруть участь.
- **Підгрупа:** Необов’язкова, для поділу на менші групи.

## 2. Множина Домени (Domains)

### Реалізація в Коді

Домени формуються функцією create_domains, що генерує можливі комбінації часу, аудиторій і викладачів:

```python
def create_domains(lessons, lecturers, auditoriums):
    domains = {}
    for lesson in lessons:
        possible_values = []
        # Фільтруємо можливих викладачів
        possible_lecturers = [lect for lect in lecturers if 
                              lesson.subject.id in lect.subjects_can_teach and 
                              lesson.type in lect.types_can_teach]
        if not possible_lecturers:
            continue  # Не має можливих викладачів, рішення неможливе

        # Фільтруємо можливі аудиторії
        group_size = lesson.group.size
        if lesson.subgroup and lesson.group.subgroups:
            group_size = math.ceil(group_size / len(lesson.group.subgroups))
        suitable_auditoriums = [aud for aud in auditoriums if aud.capacity >= group_size]
        if not suitable_auditoriums:
            continue  # Не має аудиторій з достатньою місткістю

        for day, period in TIME_SLOTS:
            for aud in suitable_auditoriums:
                for lect in possible_lecturers:
                    # Перевіряємо, чи викладач не перевищує максимальної кількості годин
                    # Це буде перевірятися під час присвоєння
                    possible_values.append((day, period, aud.id, lect.id))
        domains[lesson.id] = possible_values
    return domains
```

### Пояснення

- **Фільтрація Викладачів**: Обираються лише ті, які відповідають предмету та типу заняття.
- **Фільтрація Аудиторій**: Враховуються приміщення, здатні вмістити потрібну кількість студентів.
- **Комбінації Часу та Ресурсів**: Створюються всі можливі варіанти розподілу ресурсів.`(день, період, аудиторія, викладач)`.

Ця функція гарантує, що кожна змінна має відповідний домен, який відповідає доступним ресурсам і вимогам.

## 3. Множина Обмежень (Constraints)

### Реалізація в Коді

ОМетод is_consistent перевіряє відповідність поточного призначення умовам:

```python
def is_consistent(self, assignment, var, value, week_number):
    day, period, aud, lect = value
    # Жорсткі обмеження
    for other_var_id, other_value in assignment.items():
        other_day, other_period, other_aud, other_lect = other_value
        # Перевірка на той же час
        if day == other_day and period == other_period:
            # 1. Одна аудиторія одночасно
            if aud == other_aud:
                return False
            # 2. Один викладач одночасно
            if lect == other_lect:
                return False
            # 3. Одна група одночасно
            if self.variables[var].group.number == self.variables[other_var_id].group.number:
                # Перевірка підгруп
                if self.variables[var].subgroup and self.variables[other_var_id].subgroup:
                    if self.variables[var].subgroup == self.variables[other_var_id].subgroup:
                        return False
                else:
                    return False

    # 4. Аудиторія має достатню місткість
    group_size = self.variables[var].group.size
    if self.variables[var].subgroup and self.variables[var].group.subgroups:
        group_size = math.ceil(group_size / len(self.variables[var].group.subgroups))
    auditorium = next((a for a in self.auditoriums if a.id == aud), None)
    if auditorium and auditorium.capacity < group_size:
        return False

    # 5. Викладач не перевищує максимальну кількість годин
    hours_assigned = sum(1 for v, val in assignment.items() if val[3] == lect)
    lecturer = next((l for l in self.lecturers if l.id == lect), None)
    if lecturer and hours_assigned >= lecturer.max_hours_per_week:
        return False

    # 6. Обмеження за типом тижня (week_type)
    subject_week_type = self.variables[var].subject.week_type
    if subject_week_type == 'even' and week_number % 2 != 0:
        return False
    if subject_week_type == 'odd' and week_number % 2 != 1:
        return False
    # 'both' тижні вже дозволені

    # 7. Специфічні обмеження (наприклад, максимум занять викладача в день)
    max_lecturer_daily_hours = 3  # Приклад: максимум 3 години на день
    daily_hours = sum(1 for v, val in assignment.items()
                      if val[3] == lect and val[0] == day)
    if daily_hours >= max_lecturer_daily_hours:
        return False

    return True
```

### Пояснення

#### Жорсткі Обмеження

1. **Унікальність Аудиторії**:
   - Перевіряється, щоб одна аудиторія не використовувалась одночасно для кількох занять.
   - Якщо `aud == other_aud`, повертається `False`.

2. **Унікальність Викладача**:
   - Забезпечується, щоб один викладач не проводив кілька занять одночасно.
   - Якщо `lect == other_lect`, повертається `False`.

3. **Конфлікт Груп**:
   - Забезпечується, щоб одна група або підгрупа не мала кілька занять одночасно.
   - Якщо `self.variables[var].group.number == self.variables[other_var_id].group.number`:
     - Якщо обидві змінні мають підгрупи, перевіряється їх рівність.
     - Якщо підгрупи відсутні або не збігаються, повертається `False`.

4. **Місткість Аудиторії**:
   - Перевіряється, чи аудиторія може вмістити кількість студентів у групі або підгрупі.
   - Якщо `auditorium.capacity < group_size`, повертається `False`.

5. **Максимальна Кількість Годин Викладача**:
   - Враховується, щоб викладач не перевищував максимально допустиму кількість годин на тиждень.
   - Якщо `hours_assigned >= lecturer.max_hours_per_week`, повертається `False`.

6. **Обмеження за Типом Тижня (`week_type`)**:
   - Заняття призначаються відповідно до типу тижня (`even`, `odd`, `both`).
   - Якщо `subject_week_type == 'even'` і номер тижня непарний, повертається `False`.
   - Якщо `subject_week_type == 'odd'` і номер тижня парний, повертається `False`.
   - Заняття з `week_type == 'both'` допускаються на будь-який тип тижня.

7. **Специфічні Обмеження**:
   - Обмеження на кількість занять викладача в день (наприклад, максимум 3 години).
   - Якщо `daily_hours >= max_lecturer_daily_hours`, повертається `False`.

### Обмеження за Типом Тижня (`week_type`)

Цей компонент відповідає наданням можливості призначати заняття лише на парні, непарні або обидва типи тижнів.

```python
# 6. Обмеження за типом тижня (week_type)
subject_week_type = self.variables[var].subject.week_type
if subject_week_type == 'even' and week_number % 2 != 0:
    return False
if subject_week_type == 'odd' and week_number % 2 != 1:
    return False
# 'both' тижні вже дозволені
```

### Специфічні Обмеження

Додаткове обмеження на кількість занять викладача в день реалізовано наступним чином:

```python
# 7. Специфічні обмеження (наприклад, максимум занять викладача в день)
max_lecturer_daily_hours = 3  # Приклад: максимум 3 години на день
daily_hours = sum(1 for v, val in assignment.items()
                  if val[3] == lect and val[0] == day)
if daily_hours >= max_lecturer_daily_hours:
    return False
```

## 4. Пошук з Поверненням (Backtracking Search)

### Реалізація в Коді

Метод `backtrack` реалізує пошук з поверненням для знаходження сумісного присвоєння змінних:

```python
def backtrack(self, assignment, week_number):
    # Якщо всі змінні присвоєні, повертаємо присвоєння
    if len(assignment) == len(self.variables):
        return assignment

    # Вибір наступної змінної
    var = self.select_unassigned_variable(assignment)

    # Впорядкування значень за LCV
    ordered_values = self.order_domain_values(var, assignment)

    for value in ordered_values:
        if self.is_consistent(assignment, var.id, value, week_number):
            assignment[var.id] = value
            result = self.backtrack(assignment, week_number)
            if result:
                return result
            del assignment[var.id]
    return None
```

### Пояснення

- **Початковий Стан**: Пусте присвоєння `{}`, де жодній змінній не присвоєно значення.
- **Крок Присвоєння**: На кожному кроці вибирається змінна за допомогою евристик (MRV, Degree), а потім значення присвоюється за евристикою LCV.
- **Перевірка Сумісності**: Перевіряється, чи не порушується жодне з обмежень при присвоєнні значення.
- **Повернення**: Якщо присвоєння веде до конфлікту, відбувається повернення (backtrack).
- **Завершення**: Якщо присвоєння охоплює всі змінні, повертається повне сумісне присвоєння.

## 5. Евристики для Підвищення Ефективності Пошуку

### Реалізація в Коді

#### Евристика з Мінімальною Кількістю Решти Значень (MRV)

Вибір змінної з мінімальною кількістю можливих значень у домені:

```python
def select_unassigned_variable(self, assignment):
    # Використовуємо MRV (Minimum Remaining Values) евристику
    unassigned_vars = [v for v in self.variables if v.id not in assignment]
    # MRV
    min_domain_size = min(len(self.domains[v.id]) for v in unassigned_vars)
    mrv_vars = [v for v in unassigned_vars if len(self.domains[v.id]) == min_domain_size]
    if len(mrv_vars) == 1:
        return mrv_vars[0]
    # Ступенева евристика (degree)
    max_degree = -1
    selected_var = None
    for var in mrv_vars:
        degree = 0
        for other_var in self.variables:
            if other_var.id != var.id and self.is_neighbor(var, other_var):
                degree += 1
        if degree > max_degree:
            max_degree = degree
            selected_var = var
    return selected_var
```

#### Ступенева Евристика (Degree Heuristic)

Якщо кілька змінних мають однаковий розмір домену, вибирається змінна, яка бере участь у найбільшій кількості обмежень:

```python
# Ступенева евристика (degree)
max_degree = -1
selected_var = None
for var in mrv_vars:
    degree = 0
    for other_var in self.variables:
        if other_var.id != var.id and self.is_neighbor(var, other_var):
            degree += 1
    if degree > max_degree:
        max_degree = degree
        selected_var = var
return selected_var
```

#### Евристика з Найменш Обмежувальним Значенням (LCV)

Сортування значень домену за кількістю конфліктів, які вони можуть спричинити:

```python
def order_domain_values(self, var, assignment):
    # Евристика з найменш обмежувальним значенням (Least Constraining Value)
    def count_conflicts(value):
        day, period, aud, lect = value
        conflicts = 0
        for other_var in self.variables:
            if other_var.id in assignment:
                continue
            for other_value in self.domains[other_var.id]:
                other_day, other_period, other_aud, other_lect = other_value
                if day == other_day and period == other_period:
                    if aud == other_aud or lect == other_lect:
                        conflicts += 1
                    if self.variables[var.id].group.number == other_var.group.number:
                        if self.variables[var.id].subgroup and other_var.subgroup:
                            if self.variables[var.id].subgroup == other_var.subgroup:
                                conflicts += 1
                        else:
                            conflicts += 1
        return conflicts

    return sorted(self.domains[var.id], key=lambda value: count_conflicts(value))
```

### Пояснення

- **MRV (Minimum Remaining Values)**: Вибір змінної з найменшим розміром домену.
- **Degree Heuristic**: Вибір змінної, що впливає на найбільше інших змінних.
- **LCV (Least Constraining Value)**: Перевага значень, що спричиняють мінімум конфліктів.

## 6. Стабільність Підгруп

### Реалізація в Коді

Стабільність підгруп забезпечується шляхом фіксації підгруп при створенні змінних та перевіркою їх у методі `is_consistent`:

```python
# Генерація практичних занять
if subject.requires_subgroups and group.subgroups:
    num_practicals_per_subgroup = math.ceil(subject.num_practicals / len(group.subgroups))
    for subgroup in group.subgroups:
        for _ in range(num_practicals_per_subgroup):
            lessons.append(Lesson(lesson_id, subject, 'Практика', group, subgroup))
            lesson_id += 1
else:
    for _ in range(subject.num_practicals):
        lessons.append(Lesson(lesson_id, subject, 'Практика', group))
        lesson_id += 1
```

Перевірка консистентності підгруп:

```python
# Перевірка підгруп
if self.variables[var].subgroup and self.variables[other_var_id].subgroup:
    if self.variables[var].subgroup == self.variables[other_var_id].subgroup:
        return False
else:
    return False
```

### Пояснення

- **Генерація Підгруп**: При створенні змінних `Lesson` для практичних занять, якщо предмет потребує підгруп і вони існують, то для кожної підгрупи створюються окремі змінні.
- **Перевірка Консистентності**: Перевіряється, щоб одна підгрупа не мала кілька занять одночасно, що забезпечує стабільність підгруп протягом розкладу.

## 7. Нежорсткі Обмеження та Фітнес Функція

### Реалізація в Коді

#### Фітнес Функція

Функція `calculate_fitness` оцінює розклад за кількістю вікон (перерв) у розкладі для кожної групи:

```python
def calculate_fitness(schedule_even, schedule_odd, groups):
    fitness = 0
    # Мінімізація кількості вікон (перерв) у розкладі для кожної групи
    for group in groups:
        for week, schedule in [('even', schedule_even), ('odd', schedule_odd)]:
            daily_schedule = {day: [] for day in DAYS}
            for key, entries in schedule.items():
                day, period = key
                for entry in entries:
                    if entry['Group'] == group.number or (group.subgroups and any(entry['Group'] == f"{group.number} (Підгрупа {sg})" for sg in group.subgroups)):
                        daily_schedule[day].append(int(period))
            # Рахуємо кількість вікон
            for day, periods in daily_schedule.items():
                if not periods:
                    continue
                periods = sorted(periods)
                windows = 0
                for i in range(1, len(periods)):
                    if periods[i] - periods[i-1] > 1:
                        windows += 1
                fitness += windows
    return fitness
```

### Пояснення

- **Нежорсткі Обмеження**: Фокус на оптимізації якості розкладу через мінімізацію кількості вікон (перерв) у розкладі для кожної групи.
- **Фітнес Функція**: Визначає кількість вікон у розкладі, де менша кількість вікон означає кращий розклад. Це дозволяє оцінити та порівняти різні можливі розклади.

## 8. Обробка Специфічних Обмежень

### Реалізація в Коді

#### Обмеження на Кількість Занять Викладача в День

Додаткове обмеження на кількість занять викладача в день реалізовано наступним чином:

```python
# 7. Специфічні обмеження (наприклад, максимум занять викладача в день)
max_lecturer_daily_hours = 3  # Приклад: максимум 3 години на день
daily_hours = sum(1 for v, val in assignment.items()
                  if val[3] == lect and val[0] == day)
if daily_hours >= max_lecturer_daily_hours:
    return False
```

### Пояснення

- **Обмеження на Кількість Занять Викладача в День**: Встановлюється максимальна кількість занять, які викладач може проводити в один день. Це гарантує, що викладач не буде перевантажений.
- **Додаткові Специфічні Обмеження**: Можуть бути додані аналогічні перевірки для інших специфічних умов, наприклад, обмеження на кількість занять викладача в тиждень або унікальні вимоги до певних аудиторій чи груп.

### Підсумок
Рішення підтримує всі ключові аспекти CSP: змінні, домени, обмеження та пошукові стратегії, доповнені ефективними евристиками.