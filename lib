import 'dart:convert';
import 'package:flutter/material.dart';
import 'package:http/http.dart' as http;
import 'package:shared_preferences/shared_preferences.dart';

const api = 'https://api.opendota.com/api';
const radiant = Color(0xFF6FBF4A);
const dire = Color(0xFFD9534F);

Future<dynamic> getJson(String path) async {
  final r = await http.get(Uri.parse('$api$path'));
  if (r.statusCode != 200) throw Exception('Сервер ответил ${r.statusCode}');
  return jsonDecode(r.body);
}

int toInt(dynamic x) => int.parse('$x');

String rankName(dynamic tier) {
  if (tier == null) return 'Без ранга';
  const medals = ['', 'Herald', 'Guardian', 'Crusader', 'Archon', 'Legend', 'Ancient', 'Divine', 'Immortal'];
  final t = toInt(tier);
  final m = t ~/ 10, s = t % 10;
  if (m < 1 || m > 8) return 'Без ранга';
  return m == 8 ? 'Immortal' : '${medals[m]} $s';
}

void main() => runApp(const App());

class App extends StatelessWidget {
  const App({super.key});
  @override
  Widget build(BuildContext context) => MaterialApp(
        title: 'Dota Stats',
        debugShowCheckedModeBanner: false,
        theme: ThemeData(
          useMaterial3: true,
          brightness: Brightness.dark,
          colorSchemeSeed: radiant,
          scaffoldBackgroundColor: const Color(0xFF14181C),
        ),
        home: const StartPage(),
      );
}

class StartPage extends StatefulWidget {
  const StartPage({super.key});
  @override
  State<StartPage> createState() => _StartPageState();
}

class _StartPageState extends State<StartPage> {
  final ctrl = TextEditingController(text: 'lendy_youtube');
  List results = [];
  bool loading = false;
  String? error;

  @override
  void initState() {
    super.initState();
    _restore();
  }

  Future<void> _restore() async {
    final p = await SharedPreferences.getInstance();
    final id = p.getInt('account_id');
    if (id != null && mounted) {
      Navigator.push(context, MaterialPageRoute(builder: (_) => ProfilePage(id)));
    }
  }

  Future<void> _search() async {
    final q = ctrl.text.trim();
    if (q.isEmpty) return;
    final n = int.tryParse(q);
    if (n != null) return _open(n);
    setState(() {
      loading = true;
      error = null;
      results = [];
    });
    try {
      final data = await getJson('/search?q=${Uri.encodeQueryComponent(q)}') as List;
      setState(() {
        results = data;
        if (data.isEmpty) error = 'Никого не нашли. Проверьте ник или введите Steam ID цифрами.';
      });
    } catch (e) {
      setState(() => error = 'Не удалось загрузить: $e');
    }
    setState(() => loading = false);
  }

  Future<void> _open(int id) async {
    final p = await SharedPreferences.getInstance();
    await p.setInt('account_id', id);
    if (!mounted) return;
    Navigator.push(context, MaterialPageRoute(builder: (_) => ProfilePage(id)));
  }

  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: const Text('Найти игрока')),
        body: Padding(
          padding: const EdgeInsets.all(16),
          child: Column(children: [
            TextField(
              controller: ctrl,
              onSubmitted: (_) => _search(),
              decoration: InputDecoration(
                labelText: 'Ник или Steam ID',
                border: const OutlineInputBorder(),
                suffixIcon: IconButton(icon: const Icon(Icons.search), onPressed: _search),
              ),
            ),
            const SizedBox(height: 12),
            if (loading) const LinearProgressIndicator(),
            if (error != null) Padding(padding: const EdgeInsets.all(8), child: Text(error!)),
            Expanded(
              child: ListView(
                children: [
                  for (final r in results)
                    ListTile(
                      leading: CircleAvatar(
                        backgroundImage: r['avatarfull'] != null ? NetworkImage(r['avatarfull']) : null,
                      ),
                      title: Text('${r['personaname']}'),
                      subtitle: Text('ID ${r['account_id']}'),
                      onTap: () => _open(toInt(r['account_id'])),
                    ),
                ],
              ),
            ),
          ]),
        ),
      );
}

class ProfilePage extends StatefulWidget {
  final int id;
  const ProfilePage(this.id, {super.key});
  @override
  State<ProfilePage> createState() => _ProfilePageState();
}

class _ProfilePageState extends State<ProfilePage> {
  late Future<List<dynamic>> future;

  @override
  void initState() {
    super.initState();
    future = _load();
  }

  Future<List<dynamic>> _load() {
    final id = widget.id;
    return Future.wait([
      getJson('/players/$id'),
      getJson('/players/$id/wl'),
      getJson('/players/$id/recentMatches'),
      getJson('/players/$id/heroes'),
      getJson('/heroes'),
    ]);
  }

  @override
  Widget build(BuildContext context) => FutureBuilder<List<dynamic>>(
        future: future,
        builder: (context, snap) {
          if (snap.connectionState != ConnectionState.done) {
            return Scaffold(appBar: AppBar(), body: const Center(child: CircularProgressIndicator()));
          }
          if (snap.hasError) {
            return Scaffold(
              appBar: AppBar(),
              body: Center(
                child: Column(mainAxisSize: MainAxisSize.min, children: [
                  Text('Не удалось загрузить данные\n${snap.error}', textAlign: TextAlign.center),
                  const SizedBox(height: 12),
                  FilledButton(onPressed: () => setState(() => future = _load()), child: const Text('Повторить')),
                ]),
              ),
            );
          }
          return _content(snap.data!);
        },
      );

  Widget _content(List<dynamic> d) {
    final player = d[0] as Map;
    final wl = d[1] as Map;
    final recent = d[2] as List;
    final heroes = d[3] as List;
    final names = {for (final h in d[4] as List) toInt(h['id']): '${h['localized_name']}'};
    final profile = (player['profile'] ?? {}) as Map;

    if (profile.isEmpty) {
      return Scaffold(
        appBar: AppBar(),
        body: const Center(
          child: Padding(
            padding: EdgeInsets.all(24),
            child: Text('Профиль не найден или скрыт. В Dota 2 включите «Выставлять публичную историю матчей».',
                textAlign: TextAlign.center),
          ),
        ),
      );
    }

    return DefaultTabController(
      length: 3,
      child: Scaffold(
        appBar: AppBar(
          title: Text('${profile['personaname']}'),
          actions: [
            IconButton(icon: const Icon(Icons.refresh), onPressed: () => setState(() => future = _load())),
          ],
          bottom: const TabBar(tabs: [Tab(text: 'Обзор'), Tab(text: 'Матчи'), Tab(text: 'Герои')]),
        ),
        body: TabBarView(children: [
          _overview(player, profile, wl, recent),
          _matches(recent, names),
          _heroes(heroes, names),
        ]),
      ),
    );
  }

  Widget _overview(Map player, Map profile, Map wl, List recent) {
    final win = toInt(wl['win']), lose = toInt(wl['lose']);
    final total = win + lose;
    final rate = total == 0 ? 0.0 : win * 100 / total;
    double avg(String k) => recent.isEmpty
        ? 0
        : recent.map((m) => (m[k] as num).toDouble()).reduce((a, b) => a + b) / recent.length;

    Widget stat(String label, String value, {Color? color}) => Padding(
          padding: const EdgeInsets.symmetric(vertical: 10),
          child: Row(mainAxisAlignment: MainAxisAlignment.spaceBetween, children: [
            Text(label, style: const TextStyle(fontSize: 16)),
            Text(value, style: TextStyle(fontSize: 20, fontWeight: FontWeight.w600, color: color)),
          ]),
        );

    return ListView(padding: const EdgeInsets.all(16), children: [
      Row(children: [
        CircleAvatar(
          radius: 36,
          backgroundImage: profile['avatarfull'] != null ? NetworkImage('${profile['avatarfull']}') : null,
        ),
        const SizedBox(width: 16),
        Expanded(
          child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
            Text('${profile['personaname']}', style: const TextStyle(fontSize: 22, fontWeight: FontWeight.w700)),
            Text(rankName(player['rank_tier']), style: const TextStyle(fontSize: 16)),
          ]),
        ),
      ]),
      const SizedBox(height: 20),
      stat('Победы', '$win', color: radiant),
      stat('Поражения', '$lose', color: dire),
      stat('Винрейт', '${rate.toStringAsFixed(1)}%'),
      const Divider(height: 32),
      Text('Последние ${recent.length} матчей (в среднем)', style: const TextStyle(fontSize: 14)),
      stat('Убийства', avg('kills').toStringAsFixed(1)),
      stat('Смерти', avg('deaths').toStringAsFixed(1)),
      stat('Помощь', avg('assists').toStringAsFixed(1)),
    ]);
  }

  Widget _matches(List recent, Map<int, String> names) {
    if (recent.isEmpty) return const Center(child: Text('Нет недавних матчей'));
    return ListView.separated(
      itemCount: recent.length,
      separatorBuilder: (_, __) => const Divider(height: 1),
      itemBuilder: (_, i) {
        final m = recent[i] as Map;
        final isRadiant = toInt(m['player_slot']) < 128;
        final won = (m['radiant_win'] as bool) == isRadiant;
        final secs = toInt(m['duration']);
        final dt = DateTime.fromMillisecondsSinceEpoch(toInt(m['start_time']) * 1000);
        final date = '${dt.day.toString().padLeft(2, '0')}.${dt.month.toString().padLeft(2, '0')}';
        return ListTile(
          leading: Icon(won ? Icons.check_circle : Icons.cancel, color: won ? radiant : dire),
          title: Text(names[toInt(m['hero_id'])] ?? 'Герой ${m['hero_id']}'),
          subtitle: Text('${m['kills']}/${m['deaths']}/${m['assists']}  ·  ${secs ~/ 60}:${(secs % 60).toString().padLeft(2, '0')}'),
          trailing: Text(date),
        );
      },
    );
  }

  Widget _heroes(List heroes, Map<int, String> names) {
    final list = heroes.where((h) => toInt(h['games']) > 0).take(40).toList();
    if (list.isEmpty) return const Center(child: Text('Нет данных по героям'));
    return ListView.separated(
      itemCount: list.length,
      separatorBuilder: (_, __) => const Divider(height: 1),
      itemBuilder: (_, i) {
        final h = list[i] as Map;
        final games = toInt(h['games']), wins = toInt(h['win']);
        final rate = wins * 100 / games;
        return ListTile(
          title: Text(names[toInt(h['hero_id'])] ?? 'Герой ${h['hero_id']}'),
          subtitle: Text('$games игр'),
          trailing: Text('${rate.toStringAsFixed(0)}%',
              style: TextStyle(fontSize: 18, fontWeight: FontWeight.w600, color: rate >= 50 ? radiant : dire)),
        );
      },
    );
  }
}
