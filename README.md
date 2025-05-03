# sql

```
using Microsoft.Data.Sqlite;
using System;
using System.Collections.Generic;

// Класс для представления заметки
public class Note
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }
}

// Класс для управления базой данных заметок
public class NoteDatabase : IDisposable
{
    private string _dbPath = "notes.db";
    private SqliteConnection _connection;

    public NoteDatabase()
    {
        _connection = new SqliteConnection($"Data Source={_dbPath}");
        InitializeDatabase();
    }

    private void InitializeDatabase()
    {
        try
        {
            _connection.Open();

            var command = _connection.CreateCommand();
            command.CommandText = @"
                CREATE TABLE IF NOT EXISTS Notes (
                    Id INTEGER PRIMARY KEY AUTOINCREMENT,
                    Title TEXT NOT NULL,
                    Content TEXT
                );
            ";
            command.ExecuteNonQuery();
        }
        catch (SqliteException ex)
        {
            Console.WriteLine($"Database error during initialization: {ex.Message}");
        }
    }

    public void CreateNote(Note note)
    {
        try
        {
            using (var command = _connection.CreateCommand())
            {
                command.CommandText = @"
                    INSERT INTO Notes (Title, Content)
                    VALUES ($title, $content);
                ";
                command.Parameters.AddWithValue("$title", note.Title);
                command.Parameters.AddWithValue("$content", note.Content);
                command.ExecuteNonQuery();
            }
        }
        catch (SqliteException ex)
        {
            Console.WriteLine($"Database error creating note: {ex.Message}");
        }
    }

    public Note GetNote(int id)
    {
        try
        {
            using (var command = _connection.CreateCommand())
            {
                command.CommandText = @"
                    SELECT Id, Title, Content
                    FROM Notes
                    WHERE Id = $id;
                ";
                command.Parameters.AddWithValue("$id", id);

                using (var reader = command.ExecuteReader())
                {
                    if (reader.Read())
                    {
                        return new Note
                        {
                            Id = reader.GetInt32(0),
                            Title = reader.GetString(1),
                            Content = reader.IsDBNull(2) ? null : reader.GetString(2)
                        };
                    }
                    else
                    {
                        return null;
                    }
                }
            }
        }
        catch (SqliteException ex)
        {
            Console.WriteLine($"Database error getting note: {ex.Message}");
            return null;
        }
    }

    public List<Note> GetAllNotes()
    {
        List<Note> notes = new List<Note>();

        try
        {
            using (var command = _connection.CreateCommand())
            {
                command.CommandText = @"
                    SELECT Id, Title, Content
                    FROM Notes;
                ";

                using (var reader = command.ExecuteReader())
                {
                    while (reader.Read())
                    {
                        notes.Add(new Note
                        {
                            Id = reader.GetInt32(0),
                            Title = reader.GetString(1),
                            Content = reader.IsDBNull(2) ? null : reader.GetString(2)
                        });
                    }
                }
            }
        }
        catch (SqliteException ex)
        {
            Console.WriteLine($"Database error getting all notes: {ex.Message}");
            return new List<Note>();
        }

        return notes;
    }

    public void UpdateNote(Note note)
    {
        try
        {
            using (var command = _connection.CreateCommand())
            {
                command.CommandText = @"
                    UPDATE Notes
                    SET Title = $title,
                        Content = $content
                    WHERE Id = $id;
                ";
                command.Parameters.AddWithValue("$id", note.Id);
                command.Parameters.AddWithValue("$title", note.Title);
                command.Parameters.AddWithValue("$content", note.Content);
                command.ExecuteNonQuery();
            }
        }
        catch (SqliteException ex)
        {
            Console.WriteLine($"Database error updating note: {ex.Message}");
        }
    }

    public void DeleteNote(int id)
    {
        try
        {
            using (var command = _connection.CreateCommand())
            {
                command.CommandText = @"
                    DELETE FROM Notes
                    WHERE Id = $id;
                ";
                command.Parameters.AddWithValue("$id", id);
                command.ExecuteNonQuery();
            }
        }
        catch (SqliteException ex)
        {
            Console.WriteLine($"Database error deleting note: {ex.Message}");
        }
    }

    public void Dispose()
    {
        _connection.Close();
        _connection.Dispose();
    }
}

// Класс для пользовательского интерфейса (консольного)
public class NoteApp : IDisposable
{
    private NoteDatabase _db = new NoteDatabase();

    public void Run()
    {
        while (true)
        {
            Console.WriteLine("\nSimple Note App");
            Console.WriteLine("1. Create Note");
            Console.WriteLine("2. View Note");
            Console.WriteLine("3. View All Notes");
            Console.WriteLine("4. Update Note");
            Console.WriteLine("5. Delete Note");
            Console.WriteLine("6. Exit");

            Console.Write("Enter your choice: ");
            string choice = Console.ReadLine();

            switch (choice)
            {
                case "1":
                    CreateNote();
                    break;
                case "2":
                    ViewNote();
                    break;
                case "3":
                    ViewAllNotes();
                    break;
                case "4":
                    UpdateNote();
                    break;
                case "5":
                    DeleteNote();
                    break;
                case "6":
                    Dispose(); // Clean up resources before exiting
                    return;
                default:
                    Console.WriteLine("Invalid choice. Please try again.");
                    break;
            }
        }
    }

    private void CreateNote()
    {
        Console.Write("Enter note title: ");
        string title = Console.ReadLine();
        if (string.IsNullOrWhiteSpace(title))
        {
            Console.WriteLine("Title cannot be empty.");
            return;
        }

        Console.Write("Enter note content: ");
        string content = Console.ReadLine();

        Note newNote = new Note { Title = title, Content = content };
        _db.CreateNote(newNote);
        Console.WriteLine("Note created successfully!");
    }

    private void ViewNote()
    {
        Console.Write("Enter note ID to view: ");
        if (int.TryParse(Console.ReadLine(), out int id))
        {
            Note note = _db.GetNote(id);
            if (note != null)
            {
                Console.WriteLine($"\nNote ID: {note.Id}");
                Console.WriteLine($"Title: {note.Title}");
                Console.WriteLine($"Content: {note.Content}");
            }
            else
            {
                Console.WriteLine("Note not found.");
            }
        }
        else
        {
            Console.WriteLine("Invalid ID format.");
        }
    }

    private void ViewAllNotes()
    {
        List<Note> notes = _db.GetAllNotes();
        if (notes.Count > 0)
        {
            Console.WriteLine("\nAll Notes:");
            foreach (var note in notes)
            {
                Console.WriteLine($"ID: {note.Id}, Title: {note.Title}");
            }
        }
        else
        {
            Console.WriteLine("No notes found.");
        }
    }

    private void UpdateNote()
    {
        Console.Write("Enter note ID to update: ");
        if (int.TryParse(Console.ReadLine(), out int id))
        {
            Note note = _db.GetNote(id);
            if (note != null)
            {
                Console.Write("Enter new title (or press Enter to keep the same): ");
                string newTitle = Console.ReadLine();
                if (!string.IsNullOrEmpty(newTitle))
                {
                    note.Title = newTitle;
                }

                Console.Write("Enter new content (or press Enter to keep the same): ");
                string newContent = Console.ReadLine();
                if (!string.IsNullOrEmpty(newContent))
                {
                    note.Content = newContent;
                }

                _db.UpdateNote(note);
                Console.WriteLine("Note updated successfully!");
            }
            else
            {
                Console.WriteLine("Note not found.");
            }
        }
        else
        {
            Console.WriteLine("Invalid ID format.");
        }
    }

    private void DeleteNote()
    {
        Console.Write("Enter note ID to delete: ");
        if (int.TryParse(Console.ReadLine(), out int id))
        {
            Note note = _db.GetNote(id);
            if (note != null)
            {
                _db.DeleteNote(id);
                Console.WriteLine("Note deleted successfully!");
            }
            else
            {
                Console.WriteLine("Note not found.");
            }
        }
        else
        {
            Console.WriteLine("Invalid ID format.");
        }
    }

    public void Dispose()
    {
        _db.Dispose();
    }
}

// Точка входа в приложение
public class Program
{
    public static void Main(string[] args)
    {
        using (NoteApp app = new NoteApp())
        {
            app.Run();
        }
    }
}

```

1. Общая структура приложения:

Приложение SimpleNoteApp представляет собой консольное приложение на C#, которое позволяет пользователю создавать, просматривать, обновлять и удалять заметки, хранящиеся в локальной базе данных SQLite. Приложение состоит из нескольких классов, каждый из которых выполняет определенную задачу:

Note: Представляет собой заметку с полями Id, Title и Content.
NoteDatabase: Отвечает за управление базой данных SQLite, включая создание таблицы, добавление, чтение, обновление и удаление заметок.
NoteApp: Отвечает за пользовательский интерфейс (консольное меню) и взаимодействие с пользователем.
Program: Точка входа в приложение.

2. Классы и их назначение:

Note (Представление данных):
```
public class Note
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }
}
```
Id: Уникальный идентификатор заметки (целое число). Устанавливается базой данных автоматически.
Title: Заголовок заметки (строка).
Content: Содержание заметки (строка).
Назначение: Этот класс используется для хранения данных о заметке. Он не содержит никакой логики, кроме хранения и доступа к данным.
NoteDatabase (Управление базой данных):
```
public class NoteDatabase : IDisposable
{
    private string _dbPath = "notes.db";
    private SqliteConnection _connection;

    public NoteDatabase()
    {
        _connection = new SqliteConnection($"Data Source={_dbPath}");
        InitializeDatabase();
    }

    private void InitializeDatabase()
    {
        try
        {
            _connection.Open();
            var command = _connection.CreateCommand();
            command.CommandText = @"
                CREATE TABLE IF NOT EXISTS Notes (
                    Id INTEGER PRIMARY KEY AUTOINCREMENT,
                    Title TEXT NOT NULL,
                    Content TEXT
                );
            ";
            command.ExecuteNonQuery();
        }
        catch (SqliteException ex)
        {
            Console.WriteLine($"Database error during initialization: {ex.Message}");
        }
    }

    // ... CRUD-операции (CreateNote, GetNote, GetAllNotes, UpdateNote, DeleteNote) ...

    public void Dispose()
    {
        _connection.Close();
        _connection.Dispose();
    }
}

```
_dbPath: Приватное поле, содержащее путь к файлу базы данных SQLite (notes.db).
_connection: Приватное поле, содержащее объект SqliteConnection, представляющий соединение с базой данных.
Конструктор NoteDatabase():
Создает новое соединение с базой данных, используя путь, указанный в _dbPath.
Вызывает метод InitializeDatabase() для создания таблицы Notes, если она еще не существует.
InitializeDatabase():
Открывает соединение с базой данных.
Создает объект SqliteCommand для выполнения SQL-запроса.
Устанавливает свойство CommandText объекта SqliteCommand равным SQL-запросу для создания таблицы Notes.
Выполняет SQL-запрос с помощью метода ExecuteNonQuery().
Обрабатывает возможные исключения SqliteException и выводит сообщение об ошибке в консоль.
CRUD-операции (методы CreateNote, GetNote, GetAllNotes, UpdateNote, DeleteNote):
Каждый метод выполняет соответствующую операцию с базой данных.
Все методы используют параметризованные SQL-запросы для предотвращения SQL-инъекций.
Все методы обрабатывают возможные исключения SqliteException и выводят сообщение об ошибке в консоль.
Dispose():
Закрывает соединение с базой данных.
Освобождает ресурсы, связанные с объектом SqliteConnection.
Назначение: Этот класс инкапсулирует всю логику работы с базой данных SQLite. Он отвечает за создание, чтение, обновление и удаление заметок в базе данных.
NoteApp (Пользовательский интерфейс):
```
public class NoteApp : IDisposable
{
    private NoteDatabase _db = new NoteDatabase();

    public void Run()
    {
        while (true)
        {
            // ... Вывод меню ...
            string choice = Console.ReadLine();

            switch (choice)
            {
                // ... Обработка выбора пользователя ...
            }
        }
    }

    // ... Методы для обработки выбора пользователя (CreateNote, ViewNote, и т.д.) ...

    public void Dispose()
    {
        _db.Dispose();
    }
}
```
_db: Приватное поле, содержащее объект NoteDatabase, который используется для работы с базой данных.
Run():
Выводит меню с доступными действиями (создать, просмотреть, обновить, удалить заметку, выйти).
Считывает выбор пользователя с консоли.
Вызывает соответствующие методы для обработки выбора пользователя.
Методы для обработки выбора пользователя (CreateNote, ViewNote, GetAllNotes, UpdateNote, DeleteNote):
Каждый метод выполняет соответствующие действия на основе выбора пользователя.
Все методы взаимодействуют с объектом NoteDatabase для работы с базой данных.
Все методы выводят информацию о результате операции в консоль.
Dispose():
Вызывает метод Dispose() объекта NoteDatabase для освобождения ресурсов.
Назначение: Этот класс отвечает за взаимодействие с пользователем через консольный интерфейс. Он принимает ввод от пользователя, вызывает соответствующие методы для работы с базой данных и выводит информацию о результате операции.
Program (Точка входа):
```
public class Program
{
    public static void Main(string[] args)
    {
        using (NoteApp app = new NoteApp())
        {
            app.Run();
        }
    }
}
```
Main():
Создает экземпляр класса NoteApp с использованием блока using, что гарантирует вызов метода Dispose() при завершении работы приложения для освобождения ресурсов.
Вызывает метод Run() объекта NoteApp для запуска приложения.
Назначение: Этот класс является точкой входа в приложение. Он создает экземпляр класса NoteApp и запускает его.
3. Как все работает (поток выполнения):

При запуске приложения выполняется метод Main() класса Program.
Метод Main() создает экземпляр класса NoteApp с использованием блока using.
Конструктор класса NoteApp создает экземпляр класса NoteDatabase.
Конструктор класса NoteDatabase создает соединение с базой данных и вызывает метод InitializeDatabase() для создания таблицы Notes, если она еще не существует.
Метод Run() класса NoteApp выводит меню с доступными действиями и ожидает ввода пользователя.
В зависимости от выбора пользователя вызывается соответствующий метод для выполнения операции (создание, чтение, обновление, удаление заметки).
Методы для работы с заметками взаимодействуют с базой данных через объект NoteDatabase.
После завершения работы приложения вызывается метод Dispose() для освобождения ресурсов (закрытие соединения с базой данных).
4. Использование NuGet пакета Microsoft.Data.Sqlite:

Для работы с базой данных SQLite необходимо установить NuGet пакет Microsoft.Data.Sqlite. Этот пакет предоставляет классы и методы, необходимые для подключения к базе данных SQLite, выполнения SQL-запросов и обработки результатов.

5. Обработка исключений и проверки ввода:

Код содержит обработку исключений SqliteException в методах работы с базой данных и проверку на пустой заголовок при создании заметки. Это позволяет приложению более надежно обрабатывать ошибки и предотвращать сбои.

6. Важные моменты:

Использование параметризованных SQL-запросов: Код использует параметризованные SQL-запросы для защиты от SQL-инъекций. Это очень важно для безопасности приложения.
Реализация интерфейса IDisposable: Классы NoteDatabase и NoteApp реализуют интерфейс IDisposable для освобождения ресурсов. Это позволяет избежать утечек памяти и других проблем.
Четкое разделение ответственности: Каждый класс выполняет определенную задачу, что упрощает понимание, поддержку и расширение кода.
