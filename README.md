# SQLite in Flutter - Complete Guide

A comprehensive guide to using SQLite database in Flutter applications with the `sqflite` package.

## Table of Contents
- [Installation](#installation)
- [Database Initialization](#database-initialization)
- [Creating Tables](#creating-tables)
- [Database Operations](#database-operations)
- [Complete Example](#complete-example)
- [Best Practices](#best-practices)
- [Common Issues](#common-issues)

## Installation

Add the following dependencies to your `pubspec.yaml` file:

```yaml
dependencies:
  flutter:
    sdk: flutter
  sqflite: ^2.3.0
  path: ^1.8.3
```

Run the following command to install the packages:

```bash
flutter pub get
```

## Database Initialization

### Basic Database Setup

Create a database helper class to manage your SQLite database:

```dart
import 'dart:async';
import 'dart:io';
import 'package:path/path.dart';
import 'package:sqflite/sqflite.dart';
import 'package:path_provider/path_provider.dart';

class DatabaseHelper {
  static final DatabaseHelper _instance = DatabaseHelper._internal();
  factory DatabaseHelper() => _instance;
  DatabaseHelper._internal();

  static Database? _database;

  Future<Database> get database async {
    if (_database != null) return _database!;
    _database = await _initDatabase();
    return _database!;
  }

  Future<Database> _initDatabase() async {
    Directory documentsDirectory = await getApplicationDocumentsDirectory();
    String path = join(documentsDirectory.path, 'my_database.db');
    
    return await openDatabase(
      path,
      version: 1,
      onCreate: _onCreate,
      onUpgrade: _onUpgrade,
    );
  }

  Future<void> _onCreate(Database db, int version) async {
    // Create tables here
    await _createTables(db);
  }

  Future<void> _onUpgrade(Database db, int oldVersion, int newVersion) async {
    // Handle database upgrades here
    if (oldVersion < newVersion) {
      // Add migration logic
    }
  }
}
```

## Creating Tables

### Define Your Data Models

```dart
class User {
  final int? id;
  final String name;
  final String email;
  final int age;
  final DateTime createdAt;

  User({
    this.id,
    required this.name,
    required this.email,
    required this.age,
    required this.createdAt,
  });

  Map<String, dynamic> toMap() {
    return {
      'id': id,
      'name': name,
      'email': email,
      'age': age,
      'created_at': createdAt.millisecondsSinceEpoch,
    };
  }

  factory User.fromMap(Map<String, dynamic> map) {
    return User(
      id: map['id'],
      name: map['name'],
      email: map['email'],
      age: map['age'],
      createdAt: DateTime.fromMillisecondsSinceEpoch(map['created_at']),
    );
  }
}
```

### Create Tables

```dart
Future<void> _createTables(Database db) async {
  // Users table
  await db.execute('''
    CREATE TABLE users (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT NOT NULL,
      email TEXT UNIQUE NOT NULL,
      age INTEGER NOT NULL,
      created_at INTEGER NOT NULL
    )
  ''');

  // Posts table (example of foreign key relationship)
  await db.execute('''
    CREATE TABLE posts (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      title TEXT NOT NULL,
      content TEXT NOT NULL,
      user_id INTEGER NOT NULL,
      created_at INTEGER NOT NULL,
      FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE
    )
  ''');

  // Create indexes for better performance
  await db.execute('CREATE INDEX idx_users_email ON users (email)');
  await db.execute('CREATE INDEX idx_posts_user_id ON posts (user_id)');
}
```

## Database Operations

### Insert Operations

```dart
class UserRepository {
  final DatabaseHelper _databaseHelper = DatabaseHelper();

  Future<int> insertUser(User user) async {
    final Database db = await _databaseHelper.database;
    return await db.insert('users', user.toMap());
  }

  Future<void> insertMultipleUsers(List<User> users) async {
    final Database db = await _databaseHelper.database;
    Batch batch = db.batch();
    
    for (User user in users) {
      batch.insert('users', user.toMap());
    }
    
    await batch.commit();
  }
}
```

### Query Operations

```dart
// Get all users
Future<List<User>> getAllUsers() async {
  final Database db = await _databaseHelper.database;
  final List<Map<String, dynamic>> maps = await db.query('users');
  
  return List.generate(maps.length, (i) {
    return User.fromMap(maps[i]);
  });
}

// Get user by ID
Future<User?> getUserById(int id) async {
  final Database db = await _databaseHelper.database;
  final List<Map<String, dynamic>> maps = await db.query(
    'users',
    where: 'id = ?',
    whereArgs: [id],
  );
  
  if (maps.isNotEmpty) {
    return User.fromMap(maps.first);
  }
  return null;
}

// Search users by name
Future<List<User>> searchUsersByName(String name) async {
  final Database db = await _databaseHelper.database;
  final List<Map<String, dynamic>> maps = await db.query(
    'users',
    where: 'name LIKE ?',
    whereArgs: ['%$name%'],
  );
  
  return List.generate(maps.length, (i) {
    return User.fromMap(maps[i]);
  });
}

// Get users with pagination
Future<List<User>> getUsersWithPagination(int page, int limit) async {
  final Database db = await _databaseHelper.database;
  final List<Map<String, dynamic>> maps = await db.query(
    'users',
    orderBy: 'created_at DESC',
    limit: limit,
    offset: page * limit,
  );
  
  return List.generate(maps.length, (i) {
    return User.fromMap(maps[i]);
  });
}
```

### Update Operations

```dart
Future<int> updateUser(User user) async {
  final Database db = await _databaseHelper.database;
  return await db.update(
    'users',
    user.toMap(),
    where: 'id = ?',
    whereArgs: [user.id],
  );
}

Future<int> updateUserEmail(int userId, String newEmail) async {
  final Database db = await _databaseHelper.database;
  return await db.update(
    'users',
    {'email': newEmail},
    where: 'id = ?',
    whereArgs: [userId],
  );
}
```

### Delete Operations

```dart
Future<int> deleteUser(int id) async {
  final Database db = await _databaseHelper.database;
  return await db.delete(
    'users',
    where: 'id = ?',
    whereArgs: [id],
  );
}

Future<int> deleteAllUsers() async {
  final Database db = await _databaseHelper.database;
  return await db.delete('users');
}
```

### Raw SQL Queries

```dart
Future<List<Map<String, dynamic>>> getUsersWithPostCount() async {
  final Database db = await _databaseHelper.database;
  return await db.rawQuery('''
    SELECT u.*, COUNT(p.id) as post_count
    FROM users u
    LEFT JOIN posts p ON u.id = p.user_id
    GROUP BY u.id
    ORDER BY post_count DESC
  ''');
}

Future<int> getUserCount() async {
  final Database db = await _databaseHelper.database;
  var result = await db.rawQuery('SELECT COUNT(*) as count FROM users');
  return Sqflite.firstIntValue(result) ?? 0;
}
```

## Complete Example

Here's a complete example of how to use SQLite in a Flutter widget:

```dart
import 'package:flutter/material.dart';

class UserListScreen extends StatefulWidget {
  @override
  _UserListScreenState createState() => _UserListScreenState();
}

class _UserListScreenState extends State<UserListScreen> {
  final UserRepository _userRepository = UserRepository();
  List<User> _users = [];
  bool _isLoading = true;

  @override
  void initState() {
    super.initState();
    _loadUsers();
  }

  Future<void> _loadUsers() async {
    setState(() {
      _isLoading = true;
    });
    
    try {
      final users = await _userRepository.getAllUsers();
      setState(() {
        _users = users;
        _isLoading = false;
      });
    } catch (e) {
      setState(() {
        _isLoading = false;
      });
      // Handle error
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Error loading users: $e')),
      );
    }
  }

  Future<void> _addUser() async {
    final newUser = User(
      name: 'John Doe',
      email: 'john@example.com',
      age: 25,
      createdAt: DateTime.now(),
    );
    
    try {
      await _userRepository.insertUser(newUser);
      _loadUsers(); // Refresh the list
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Error adding user: $e')),
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Users'),
        actions: [
          IconButton(
            icon: Icon(Icons.add),
            onPressed: _addUser,
          ),
        ],
      ),
      body: _isLoading
          ? Center(child: CircularProgressIndicator())
          : ListView.builder(
              itemCount: _users.length,
              itemBuilder: (context, index) {
                final user = _users[index];
                return ListTile(
                  title: Text(user.name),
                  subtitle: Text(user.email),
                  trailing: Text('Age: ${user.age}'),
                  onTap: () {
                    // Handle user tap
                  },
                );
              },
            ),
    );
  }
}
```

## Best Practices

### 1. Use Transactions for Multiple Operations
```dart
Future<void> performMultipleOperations() async {
  final Database db = await _databaseHelper.database;
  
  await db.transaction((txn) async {
    await txn.insert('users', user1.toMap());
    await txn.insert('posts', post1.toMap());
    await txn.update('users', user2.toMap(), where: 'id = ?', whereArgs: [user2.id]);
  });
}
```

### 2. Handle Database Migrations
```dart
Future<void> _onUpgrade(Database db, int oldVersion, int newVersion) async {
  if (oldVersion < 2) {
    await db.execute('ALTER TABLE users ADD COLUMN phone TEXT');
  }
  if (oldVersion < 3) {
    await db.execute('ALTER TABLE users ADD COLUMN avatar_url TEXT');
  }
}
```

### 3. Use Proper Error Handling
```dart
Future<List<User>> getAllUsers() async {
  try {
    final Database db = await _databaseHelper.database;
    final List<Map<String, dynamic>> maps = await db.query('users');
    return List.generate(maps.length, (i) => User.fromMap(maps[i]));
  } catch (e) {
    print('Error getting users: $e');
    return [];
  }
}
```

### 4. Close Database Connections
```dart
Future<void> closeDatabase() async {
  final Database db = await _databaseHelper.database;
  await db.close();
}
```

## Common Issues

### Issue 1: Database Lock
**Problem**: Database is locked error
**Solution**: Use transactions properly and avoid concurrent operations

### Issue 2: Table Doesn't Exist
**Problem**: No such table error
**Solution**: Ensure tables are created in `_onCreate` method

### Issue 3: Column Constraints
**Problem**: Constraint violation errors
**Solution**: Handle unique constraints and foreign keys properly

### Issue 4: Data Type Mismatches
**Problem**: Type conversion errors
**Solution**: Use proper data types in your model classes

## Advanced Features

### Database Backup and Restore
```dart
Future<void> backupDatabase() async {
  final Database db = await _databaseHelper.database;
  final String dbPath = db.path;
  
  // Copy database file to external storage
  final Directory appDocDir = await getApplicationDocumentsDirectory();
  final File backupFile = File('${appDocDir.path}/backup.db');
  await File(dbPath).copy(backupFile.path);
}
```

### Database Encryption (Optional)
For sensitive data, consider using encrypted databases with packages like `sqlite3_flutter_libs` with encryption support.

## Resources

- [sqflite Package Documentation](https://pub.dev/packages/sqflite)
- [SQLite Official Documentation](https://www.sqlite.org/docs.html)
- [Flutter Database Guide](https://docs.flutter.dev/cookbook/persistence/sqlite)

## Contributing

Feel free to contribute to this guide by submitting pull requests or opening issues for improvements and corrections.
