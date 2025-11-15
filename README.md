// KnowCredit - Flutter Starter (single-file prototype)
// File: lib/main.dart
// Purpose: Prototype of core flows described by the user:
//  - Seller enters buyer PAN & amount
//  - OTP approval (simulated) by buyer
//  - Store transactions locally (SharedPreferences JSON)
//  - View PAN profile (outstanding only)
//  - Seller can mark payment received
//  - Reward points for timely repayment (within financial year)
// Notes for production: replace simulated OTP with real SMS/OTP service (e.g., Twilio), replace SharedPreferences with secure DB (Hive/SQLite), add authentication for sellers, encrypt local storage, and integrate backend for cross-device syncing.

import 'dart:convert';
import 'dart:math';
import 'package:flutter/material.dart';
import 'package:shared_preferences/shared_preferences.dart';

void main() {
  runApp(KnowCreditApp());
}

class KnowCreditApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'KnowCredit (Prototype)',
      theme: ThemeData(primarySwatch: Colors.blue),
      home: HomePage(),
    );
  }
}

// Data models (simple maps saved as JSON in SharedPreferences)
// Transaction example stored per PAN:
// { "pan": "ABCDE1234F", "transactions": [ {"id": "t1", "seller": "Y Store", "amount": 500, "date":"2025-11-16", "paid": false} ], "rewardPoints": 0, "score": 100 }

class HomePage extends StatefulWidget {
  @override
  _HomePageState createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  final _panController = TextEditingController();
  final _amountController = TextEditingController();
  final _sellerController = TextEditingController();

  Map<String, dynamic> db = {}; // pan -> profile map

  @override
  void initState() {
    super.initState();
    _loadDb();
  }

  Future<void> _loadDb() async {
    final sp = await SharedPreferences.getInstance();
    final raw = sp.getString('knowcredit_db') ?? '{}';
    setState(() {
      db = jsonDecode(raw);
    });
  }

  Future<void> _saveDb() async {
    final sp = await SharedPreferences.getInstance();
    await sp.setString('knowcredit_db', jsonEncode(db));
  }

  String _generateId() => DateTime.now().millisecondsSinceEpoch.toString();

  // Simulate sending OTP. In production use SMS provider.
  String _generateOtp() {
    final rnd = Random();
    return (100000 + rnd.nextInt(900000)).toString();
  }

  void _startCreditFlow() async {
    final pan = _panController.text.trim().toUpperCase();
    final seller = _sellerController.text.trim();
    final amountText = _amountController.text.trim();
    if (pan.isEmpty || seller.isEmpty || amountText.isEmpty) {
      _showMessage('Please fill PAN, Seller and Amount');
      return;
    }
    final amount = double.tryParse(amountText);
    if (amount == null || amount <= 0) {
      _showMessage('Enter a valid amount');
      return;
    }

    final otp = _generateOtp();

    // For prototype: show OTP in dialog (simulated receiving by buyer)
    final approved = await showDialog<bool>(
      context: context,
      builder: (ctx) => OtpDialog(pan: pan, otp: otp),
    );

    if (approved != true) {
      _showMessage('Transaction not approved by buyer');
      return;
    }

    // save transaction
    final tx = {
      'id': _generateId(),
      'seller': seller,
      'amount': amount,
      'date': DateTime.now().toIso8601String(),
      'paid': false
    };

    if (!db.containsKey(pan)) {
      db[pan] = {
        'pan': pan,
        'transactions': [tx],
        'rewardPoints': 0,
        'score': 100
      };
    } else {
      final list = List.from(db[pan]['transactions'] ?? []);
      list.add(tx);
      db[pan]['transactions'] = list;
    }

    await _saveDb();
    await _loadDb();
    _showMessage('Credit recorded for $pan (₹${amount.toStringAsFixed(0)})');
  }

  void _showMessage(String t) {
    ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(t)));
  }

  void _openProfile(String pan) {
    Navigator.of(context).push(MaterialPageRoute(builder: (_) => ProfilePage(pan: pan)));
  }

  @override
  Widget build(BuildContext context) {
    final pans = db.keys.toList()..sort();
    return Scaffold(
      appBar: AppBar(title: Text('KnowCredit - Prototype')),
      body: Padding(
        padding: const EdgeInsets.all(12.0),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('Record new credit (Seller side)', style: TextStyle(fontWeight: FontWeight.bold)),
            SizedBox(height: 8),
            TextField(controller: _sellerController, decoration: InputDecoration(labelText: 'Seller name (you)')),
            SizedBox(height: 8),
            TextField(controller: _panController, decoration: InputDecoration(labelText: 'Buyer PAN (e.g. ABCDE1234F)')),
            SizedBox(height: 8),
            TextField(controller: _amountController, decoration: InputDecoration(labelText: 'Amount (₹)'), keyboardType: TextInputType.number),
            SizedBox(height: 8),
            ElevatedButton(onPressed: _startCreditFlow, child: Text('Send OTP & Record Credit')),
            Divider(height: 20),
            Text('Lookup PAN profiles', style: TextStyle(fontWeight: FontWeight.bold)),
            SizedBox(height: 8),
            Expanded(
              child: pans.isEmpty
                  ? Center(child: Text('No PAN profiles yet'))
                  : ListView.builder(
                      itemCount: pans.length,
                      itemBuilder: (ctx, i) {
                        final pan = pans[i];
                        final profile = db[pan];
                        final totalOutstanding = (profile['transactions'] as List)
                            .where((t) => t['paid'] == false)
                            .fold(0.0, (s, t) => s + (t['amount'] as num).toDouble());
                        return ListTile(
                          title: Text(pan),
                          subtitle: Text('Outstanding: ₹${totalOutstanding.toStringAsFixed(0)}'),
                          trailing: Icon(Icons.chevron_right),
                          onTap: () => _openProfile(pan),
                        );
                      },
                    ),
            )
          ],
        ),
      ),
    );
  }
}

class OtpDialog extends StatefulWidget {
  final String pan;
  final String otp;
  OtpDialog({required this.pan, required this.otp});

  @override
  _OtpDialogState createState() => _OtpDialogState();
}

class _OtpDialogState extends State<OtpDialog> {
  final _input = TextEditingController();
  bool _showOtpForDev = false; // toggle to reveal OTP for prototype

  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      title: Text('OTP verification (simulated)'),
      content: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          Text('Buyer PAN: ${widget.pan}'),
          SizedBox(height: 8),
          Text('Enter OTP sent to buyer\'s phone'),
          TextField(controller: _input, keyboardType: TextInputType.number),
          SizedBox(height: 8),
          if (_showOtpForDev) Text('Simulated OTP: ${widget.otp}', style: TextStyle(fontWeight: FontWeight.bold)),
          Row(children: [
            Checkbox(value: _showOtpForDev, onChanged: (v) { setState(() { _showOtpForDev = v ?? false; }); }),
            Expanded(child: Text('Show OTP here (development only)'))
          ])
        ],
      ),
      actions: [
        TextButton(onPressed: () => Navigator.of(context).pop(false), child: Text('Cancel')),
        TextButton(
            onPressed: () {
              if (_input.text.trim() == widget.otp) {
                Navigator.of(context).pop(true);
              } else {
                ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('Wrong OTP')));
              }
            },
            child: Text('Approve'))
      ],
    );
  }
}

class ProfilePage extends StatefulWidget {
  final String pan;
  ProfilePage({required this.pan});
  @override
  _ProfilePageState createState() => _ProfilePageState();
}

class _ProfilePageState extends State<ProfilePage> {
  Map<String, dynamic> db = {};

  @override
  void initState() {
    super.initState();
    _loadDb();
  }

  Future<void> _loadDb() async {
    final sp = await SharedPreferences.getInstance();
    final raw = sp.getString('knowcredit_db') ?? '{}';
    setState(() { db = jsonDecode(raw); });
  }

  Future<void> _saveDb() async {
    final sp = await SharedPreferences.getInstance();
    await sp.setString('knowcredit_db', jsonEncode(db));
  }

  double _totalOutstanding(List txs) {
    return txs.where((t) => t['paid'] == false).fold(0.0, (s, t) => s + (t['amount'] as num).toDouble());
  }

  Future<void> _markPaid(String txId) async {
    final profile = db[widget.pan];
    if (profile == null) return;
    final list = List.from(profile['transactions']);
    final idx = list.indexWhere((t) => t['id'] == txId);
    if (idx == -1) return;
    list[idx]['paid'] = true;
    profile['transactions'] = list;

    // reward if cleared within financial year
    final txDate = DateTime.parse(list[idx]['date']);
    final clearedDate = DateTime.now();
    if (_isSameFinancialYear(txDate, clearedDate)) {
      profile['rewardPoints'] = (profile['rewardPoints'] ?? 0) + 10; // arbitrary reward
    }

    // recalc score: simple rule - if outstanding at financial year end, drop score
    profile['score'] = _recalculateScore(profile);

    db[widget.pan] = profile;
    await _saveDb();
    await _loadDb();
  }

  bool _isSameFinancialYear(DateTime a, DateTime b) {
    // Indian financial year: April 1 - March 31
    int fyStartYear(DateTime d) => (d.month >= 4) ? d.year : d.year - 1;
    return fyStartYear(a) == fyStartYear(b);
  }

  int _recalculateScore(Map profile) {
    final txs = List.from(profile['transactions'] ?? []);
    final outstanding = txs.where((t) => t['paid'] == false).fold(0.0, (s, t) => s + (t['amount'] as num).toDouble());
    int base = 100;
    if (outstanding == 0) return base;
    // simple linear penalty: more outstanding -> lower score
    final penalty = (outstanding / 1000).clamp(0, 80); // every ₹1000 reduces some points
    return (base - penalty).round();
  }

  Future<void> _deleteProfile() async {
    db.remove(widget.pan);
    await _saveDb();
    Navigator.of(context).pop();
  }

  @override
  Widget build(BuildContext context) {
    final profile = db[widget.pan];
    if (profile == null) {
      return Scaffold(appBar: AppBar(title: Text(widget.pan)), body: Center(child: Text('No profile')));
    }
    final txs = List.from(profile['transactions'] ?? [])..sort((a,b)=> a['paid']?1:-1);
    final outstanding = _totalOutstanding(txs);
    final rewardPoints = profile['rewardPoints'] ?? 0;
    final score = profile['score'] ?? 100;

    return Scaffold(
      appBar: AppBar(title: Text('Profile: ${widget.pan}')),
      body: Padding(
        padding: const EdgeInsets.all(12.0),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('Outstanding: ₹${outstanding.toStringAsFixed(0)}', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
            SizedBox(height: 6),
            Text('Reward points: $rewardPoints    |    KnowCredit Score: $score'),
            Divider(),
            Text('Transactions', style: TextStyle(fontWeight: FontWeight.bold)),
            SizedBox(height: 8),
            Expanded(
              child: txs.isEmpty
                  ? Center(child: Text('No transactions'))
                  : ListView.builder(
                      itemCount: txs.length,
                      itemBuilder: (ctx, i) {
                        final t = txs[i];
                        return Card(
                          child: ListTile(
                            title: Text('${t['seller']} — ₹${(t['amount'] as num).toStringAsFixed(0)}'),
                            subtitle: Text('Date: ${DateTime.parse(t['date']).toLocal().toString().split(' ')[0]}'),
                            trailing: t['paid'] == true
                                ? Icon(Icons.check, color: Colors.green)
                                : ElevatedButton(
                                    onPressed: () async {
                                      // seller marks as received
                                      final confirm = await showDialog<bool>(context: context, builder: (c)=> AlertDialog(
                                        title: Text('Mark paid?'),
                                        content: Text('Mark ₹${(t['amount'] as num).toStringAsFixed(0)} paid to ${t['seller']}?'),
                                        actions: [TextButton(onPressed: ()=>Navigator.of(c).pop(false), child: Text('No')), TextButton(onPressed: ()=>Navigator.of(c).pop(true), child: Text('Yes'))],
                                      ));
                                      if (confirm==true) {
                                        await _markPaid(t['id']);
                                      }
                                    },
                                    child: Text('Mark Received'),
                                  ),
                          ),
                        );
                      },
                    ),
            ),
            ElevatedButton(onPressed: _deleteProfile, child: Text('Delete profile (dev only)'))
          ],
        ),
      ),
    );
  }
}
