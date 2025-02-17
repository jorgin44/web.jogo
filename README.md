import 'package:flutter/material.dart';
import 'package:sqflite/sqflite.dart';
import 'package:path/path.dart';

void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      title: 'CRUD de Planetas',
      home: PlanetListScreen(),
    );
  }
}

class DatabaseHelper {
  static final DatabaseHelper _instance = DatabaseHelper._internal();
  factory DatabaseHelper() => _instance;
  DatabaseHelper._internal();

  Database? _database;

  Future<Database> get database async {
    if (_database != null) return _database!;
    _database = await _initDatabase();
    return _database!;
  }

  Future<Database> _initDatabase() async {
    String path = join(await getDatabasesPath(), 'planets.db');
    return await openDatabase(
      path,
      version: 1,
      onCreate: (db, version) async {
        await db.execute('''
          CREATE TABLE planets (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            distance REAL NOT NULL,
            size INTEGER NOT NULL,
            nickname TEXT
          )
        ''');
      },
    );
  }

  Future<List<Map<String, dynamic>>> getPlanets() async {
    final db = await database;
    return await db.query('planets');
  }

  Future<int> insertPlanet(Map<String, dynamic> planet) async {
    final db = await database;
    return await db.insert('planets', planet);
  }

  Future<int> updatePlanet(Map<String, dynamic> planet) async {
    final db = await database;
    return await db.update('planets', planet, where: 'id = ?', whereArgs: [planet['id']]);
  }

  Future<int> deletePlanet(int id) async {
    final db = await database;
    return await db.delete('planets', where: 'id = ?', whereArgs: [id]);
  }
}

class PlanetListScreen extends StatefulWidget {
  @override
  _PlanetListScreenState createState() => _PlanetListScreenState();
}

class _PlanetListScreenState extends State<PlanetListScreen> {
  final DatabaseHelper dbHelper = DatabaseHelper();
  List<Map<String, dynamic>> planets = [];

  void refreshPlanets() async {
    final data = await dbHelper.getPlanets();
    setState(() {
      planets = data;
    });
  }

  @override
  void initState() {
    super.initState();
    refreshPlanets();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Planetas')),
      body: ListView.builder(
        itemCount: planets.length,
        itemBuilder: (context, index) {
          final planet = planets[index];
          return ListTile(
            title: Text(planet['name']),
            subtitle: Text('Distância: ${planet['distance']} UA, Tamanho: ${planet['size']} km'),
            trailing: IconButton(
              icon: Icon(Icons.delete, color: Colors.red),
              onPressed: () {
                dbHelper.deletePlanet(planet['id']);
                refreshPlanets();
              },
            ),
            onTap: () async {
              await Navigator.push(
                context,
                MaterialPageRoute(
                  builder: (context) => PlanetFormScreen(planet: planet, refresh: refreshPlanets),
                ),
              );
              refreshPlanets();
            },
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        child: Icon(Icons.add),
        onPressed: () async {
          await Navigator.push(
            context,
            MaterialPageRoute(
              builder: (context) => PlanetFormScreen(refresh: refreshPlanets),
            ),
          );
          refreshPlanets();
        },
      ),
    );
  }
}

class PlanetFormScreen extends StatefulWidget {
  final Map<String, dynamic>? planet;
  final VoidCallback refresh;
  
  PlanetFormScreen({this.planet, required this.refresh});

  @override
  _PlanetFormScreenState createState() => _PlanetFormScreenState();
}

class _PlanetFormScreenState extends State<PlanetFormScreen> {
  final _formKey = GlobalKey<FormState>();
  String name = '';
  double distance = 0.0;
  int size = 0;
  String? nickname;
  final DatabaseHelper dbHelper = DatabaseHelper();

  @override
  void initState() {
    super.initState();
    if (widget.planet != null) {
      name = widget.planet!['name'];
      distance = widget.planet!['distance'];
      size = widget.planet!['size'];
      nickname = widget.planet!['nickname'];
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(widget.planet == null ? 'Adicionar Planeta' : 'Editar Planeta')),
      body: Padding(
        padding: EdgeInsets.all(16.0),
        child: Form(
          key: _formKey,
          child: Column(
            children: [
              TextFormField(
                initialValue: name,
                decoration: InputDecoration(labelText: 'Nome'),
                validator: (value) => value!.isEmpty ? 'Campo obrigatório' : null,
                onChanged: (value) => name = value,
              ),
              TextFormField(
                initialValue: distance.toString(),
                decoration: InputDecoration(labelText: 'Distância do Sol (UA)'),
                keyboardType: TextInputType.number,
                onChanged: (value) => distance = double.parse(value),
              ),
              TextFormField(
                initialValue: size.toString(),
                decoration: InputDecoration(labelText: 'Tamanho (km)'),
                keyboardType: TextInputType.number,
                onChanged: (value) => size = int.parse(value),
              ),
              TextFormField(
                initialValue: nickname,
                decoration: InputDecoration(labelText: 'Apelido (Opcional)'),
                onChanged: (value) => nickname = value,
              ),
              SizedBox(height: 20),
              ElevatedButton(
                onPressed: () async {
                  if (_formKey.currentState!.validate()) {
                    final planet = {'name': name, 'distance': distance, 'size': size, 'nickname': nickname};
                    if (widget.planet == null) {
                      await dbHelper.insertPlanet(planet);
                    } else {
                      planet['id'] = widget.planet!['id'];
                      await dbHelper.updatePlanet(planet);
                    }
                    widget.refresh();
                    Navigator.pop(context);
                  }
                },
                child: Text(widget.planet == null ? 'Adicionar' : 'Atualizar'),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
