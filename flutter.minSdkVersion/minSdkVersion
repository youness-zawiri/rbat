name: Build APK
on:
  workflow_dispatch:
  push:
    branches: [main, master]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
      - uses: subosito/flutter-action@v2
        with:
          channel: stable
      - name: Build
        run: |
          flutter create --project-name rabat_truck --org com.mapx --platforms android .
          rm -rf test
          sed -i 's/android:label="[^"]*"/android:label="شاحنة الرباط"/' android/app/src/main/AndroidManifest.xml
          cat > lib/main.dart << 'EOF'
          import 'dart:math';
          import 'package:flutter/material.dart';
          import 'package:flutter/scheduler.dart';
          import 'package:flutter/services.dart';

          void main() {
            WidgetsFlutterBinding.ensureInitialized();
            SystemChrome.setPreferredOrientations([DeviceOrientation.portraitUp]);
            SystemChrome.setEnabledSystemUIMode(SystemUiMode.immersiveSticky);
            runApp(const MaterialApp(
              debugShowCheckedModeBanner: false,
              home: Directionality(textDirection: TextDirection.rtl, child: Game()),
            ));
          }

          const landmarks = ['برج حسان', 'قصبة الأوداية', 'ضريح محمد الخامس', 'موقع شالة'];

          // type: 0 طاكسي صغير، 1 طاكسي كبير، 2 حافلة، 3 حاجز، 4 طرد
          class Obj {
            int lane;
            double y;
            int type;
            double rel;
            Obj(this.lane, this.y, this.type, this.rel);
          }

          class Game extends StatefulWidget {
            const Game({super.key});
            @override
            State<Game> createState() => GameState();
          }

          class GameState extends State<Game> with SingleTickerProviderStateMixin {
            late final Ticker ticker;
            Duration last = Duration.zero;
            final rnd = Random();
            Size size = Size.zero;
            int state = 0; // 0 قائمة، 1 لعب، 2 نهاية
            double lane = 1, speed = 0, dist = 0, scroll = 0, inv = 0;
            double spawnT = 1, toast = 0, lmY = -9999, lmNext = 250;
            int target = 1, lives = 3, boxes = 0, best = 0, lmIdx = 0, lmKind = 0;
            String toastText = '';
            List<Obj> objs = [];

            double get roadW => size.width * 0.64;
            double get roadL => (size.width - roadW) / 2;
            double get laneW => roadW / 3;
            double get truckY => size.height * 0.78;
            double laneX(double l) => roadL + laneW * (l + 0.5);
            int get score => boxes * 50 + dist.floor();

            double objW(int t) => [0.62, 0.62, 0.66, 0.4, 0.45][t] * laneW;
            double objH(int t) => [1.25, 1.25, 2.0, 0.4, 0.45][t] * laneW;

            @override
            void initState() {
              super.initState();
              ticker = createTicker(tick)..start();
            }

            @override
            void dispose() {
              ticker.dispose();
              super.dispose();
            }

            void startGame() {
              setState(() {
                lane = 1;
                target = 1;
                dist = 0;
                scroll = 0;
                lives = 3;
                boxes = 0;
                inv = 0;
                spawnT = 1;
                objs = [];
                lmIdx = 0;
                lmNext = 250;
                lmY = -9999;
                toast = 0;
                state = 1;
              });
            }

            void tick(Duration el) {
              final dt = min((el - last).inMicroseconds / 1e6, 0.05);
              last = el;
              if (size == Size.zero) return;
              if (state == 1) {
                update(dt);
              } else {
                scroll += size.height * 0.15 * dt;
              }
              setState(() {});
            }

            void spawn() {
              final l = rnd.nextInt(3);
              final r = rnd.nextDouble();
              if (r < 0.28) {
                objs.add(Obj(l, -laneW * 2, 4, 1.0));
                return;
              }
              int t;
              double rel;
              if (r < 0.58) {
                t = 0;
                rel = 0.55;
              } else if (r < 0.75) {
                t = 1;
                rel = 0.6;
              } else if (r < 0.88) {
                t = 2;
                rel = 0.45;
              } else {
                t = 3;
                rel = 1.0;
              }
              objs.add(Obj(l, -laneW * 2.5, t, rel));
              if (dist > 400 && rnd.nextDouble() < 0.3) {
                final l2 = (l + 1 + rnd.nextInt(2)) % 3;
                objs.add(Obj(l2, -laneW * 2.5 - rnd.nextDouble() * laneW, 0, 0.55));
              }
            }

            bool hit(Obj o) {
              final tw = laneW * 0.6 * 0.8;
              final th = laneW * 1.9 * 0.85;
              final ox = laneX(o.lane.toDouble());
              final ow = objW(o.type) * 0.85;
              final oh = objH(o.type) * 0.85;
              return (laneX(lane) - ox).abs() < (tw + ow) / 2 &&
                  (truckY - o.y).abs() < (th + oh) / 2;
            }

            void update(double dt) {
              final h = size.height;
              speed = h * (0.55 + min(dist / 3000, 0.6));
              dist += speed * dt * 60 / h;
              scroll += speed * dt;
              lane += (target - lane) * min(1.0, dt * 12);
              inv = max(0.0, inv - dt);
              toast = max(0.0, toast - dt);
              spawnT -= dt;
              if (spawnT <= 0) {
                spawn();
                spawnT = (0.45 + rnd.nextDouble() * 0.6) * (1 - min(dist / 6000, 0.45));
              }
              for (final o in objs) {
                o.y += speed * dt * o.rel;
              }
              lmY += speed * dt;
              if (dist >= lmNext) {
                lmKind = lmIdx % landmarks.length;
                lmIdx++;
                lmNext += 350;
                lmY = -200;
                toastText = 'أنت الآن عند: ${landmarks[lmKind]}';
                toast = 3;
              }
              final remove = <Obj>[];
              for (final o in objs) {
                if (o.y > h + laneW * 3) {
                  remove.add(o);
                  continue;
                }
                if (hit(o)) {
                  remove.add(o);
                  if (o.type == 4) {
                    boxes++;
                    HapticFeedback.lightImpact();
                  } else if (inv <= 0) {
                    lives--;
                    inv = 1.5;
                    HapticFeedback.heavyImpact();
                    if (lives <= 0) {
                      state = 2;
                      best = max(best, score);
                    }
                  }
                }
              }
              objs.removeWhere(remove.contains);
            }

            static const hudStyle = TextStyle(
              color: Colors.white,
              fontSize: 20,
              fontWeight: FontWeight.bold,
              shadows: [Shadow(blurRadius: 4, color: Colors.black)],
            );

            Widget hud() => Align(
                  alignment: Alignment.topCenter,
                  child: SafeArea(
                    child: Padding(
                      padding: const EdgeInsets.all(12),
                      child: Row(
                        mainAxisAlignment: MainAxisAlignment.spaceBetween,
                        children: [
                          Text('❤️' * lives, style: const TextStyle(fontSize: 22)),
                          Text('📦 $boxes', style: hudStyle),
                          Text('$score م', style: hudStyle),
                        ],
                      ),
                    ),
                  ),
                );

            Widget panel(String title, String sub, String btn) => Container(
                  color: Colors.black54,
                  child: Center(
                    child: Container(
                      margin: const EdgeInsets.all(28),
                      padding: const EdgeInsets.all(24),
                      decoration: BoxDecoration(
                        color: Colors.white,
                        borderRadius: BorderRadius.circular(24),
                      ),
                      child: Column(
                        mainAxisSize: MainAxisSize.min,
                        children: [
                          Text(title,
                              textAlign: TextAlign.center,
                              style: const TextStyle(
                                  fontSize: 30,
                                  fontWeight: FontWeight.bold,
                                  color: Color(0xFFC1272D))),
                          const SizedBox(height: 12),
                          Text(sub,
                              textAlign: TextAlign.center,
                              style: const TextStyle(fontSize: 16, color: Colors.black87)),
                          const SizedBox(height: 20),
                          FilledButton(
                            onPressed: startGame,
                            child: Text(btn, style: const TextStyle(fontSize: 20)),
                          ),
                        ],
                      ),
                    ),
                  ),
                );

            @override
            Widget build(BuildContext context) {
              return Scaffold(
                backgroundColor: Colors.black,
                body: LayoutBuilder(builder: (context, cons) {
                  size = Size(cons.maxWidth, cons.maxHeight);
                  return Stack(children: [
                    Positioned.fill(
                      child: GestureDetector(
                        behavior: HitTestBehavior.opaque,
                        onTapDown: (d) {
                          if (state != 1) return;
                          final left = d.localPosition.dx < size.width / 2;
                          target = (target + (left ? -1 : 1)).clamp(0, 2).toInt();
                        },
                        child: CustomPaint(painter: Painter(this)),
                      ),
                    ),
                    if (state == 1) hud(),
                    if (state == 1 && toast > 0)
                      Positioned(
                        top: size.height * 0.14,
                        left: 20,
                        right: 20,
                        child: Center(
                          child: Container(
                            padding:
                                const EdgeInsets.symmetric(horizontal: 18, vertical: 10),
                            decoration: BoxDecoration(
                              color: const Color(0xFF1B8A4B),
                              borderRadius: BorderRadius.circular(20),
                            ),
                            child: Text(toastText,
                                style: const TextStyle(
                                    color: Colors.white,
                                    fontSize: 18,
                                    fontWeight: FontWeight.bold)),
                          ),
                        ),
                      ),
                    if (state == 0)
                      panel('🚚 شاحنة الرباط',
                          'وصّل الطرود عبر معالم الرباط وتفادَ السيارات\n\nاضغط يميناً أو يساراً لتغيير المسار', 'ابدأ'),
                    if (state == 2)
                      panel('انتهت الرحلة',
                          'المسافة والنقاط: $score\nالطرود: $boxes\nأفضل نتيجة: $best', 'العب مرة أخرى'),
                  ]);
                }),
              );
            }
          }

          class Painter extends CustomPainter {
            final GameState g;
            Painter(this.g);

            @override
            bool shouldRepaint(covariant CustomPainter oldDelegate) => true;

            @override
            void paint(Canvas canvas, Size s) {
              final w = s.width, h = s.height;
              final rl = g.roadL, rw = g.roadW, lw = g.laneW;
              final p = Paint();
              p.color = const Color(0xFFE9DDBB);
              canvas.drawRect(Rect.fromLTWH(0, 0, w, h), p);
              sideDecor(canvas, s);
              p.color = const Color(0xFF3A3F47);
              canvas.drawRect(Rect.fromLTWH(rl, 0, rw, h), p);
              p.color = Colors.white;
              canvas.drawRect(Rect.fromLTWH(rl - 3, 0, 4, h), p);
              canvas.drawRect(Rect.fromLTWH(rl + rw - 1, 0, 4, h), p);
              final off = g.scroll % 80;
              for (int i = 1; i < 3; i++) {
                for (double y = -80 + off; y < h; y += 80) {
                  canvas.drawRect(Rect.fromLTWH(rl + lw * i - 2, y, 4, 40), p);
                }
              }
              landmark(canvas, s);
              for (final o in g.objs) {
                drawObj(canvas, o);
              }
              truck(canvas);
            }

            void sideDecor(Canvas c, Size s) {
              final w = s.width, h = s.height;
              final sideW = g.roadL;
              const gap = 150.0;
              final n = (h / gap).ceil() + 2;
              for (int k = 0; k < n; k++) {
                final y = ((g.scroll + k * gap) % (n * gap)) - gap;
                if (k.isEven) {
                  building(c, Rect.fromLTWH(sideW * 0.1, y, sideW * 0.8, 90));
                  palm(c, Offset(w - sideW * 0.5, y + 40));
                } else {
                  palm(c, Offset(sideW * 0.5, y + 40));
                  building(c, Rect.fromLTWH(w - sideW * 0.9, y, sideW * 0.8, 90));
                }
              }
            }

            void building(Canvas c, Rect r) {
              final p = Paint()..color = Colors.white;
              c.drawRRect(RRect.fromRectAndRadius(r, const Radius.circular(4)), p);
              p.color = const Color(0xFF5B9BD5);
              final cw = (r.width - 12) / 3;
              for (int i = 0; i < 3; i++) {
                for (int j = 0; j < 4; j++) {
                  c.drawRect(
                      Rect.fromLTWH(r.left + 6 + i * cw, r.top + 8 + j * 20, cw - 4, 10),
                      p);
                }
              }
            }

            void palm(Canvas c, Offset o) {
              final p = Paint()
                ..color = const Color(0xFF2E8B3D)
                ..strokeWidth = 5
                ..strokeCap = StrokeCap.round;
              for (int i = 0; i < 7; i++) {
                final a = i * pi * 2 / 7;
                c.drawLine(o, o + Offset(cos(a), sin(a)) * 18, p);
              }
              p.color = const Color(0xFF7A4A21);
              c.drawCircle(o, 5, p);
            }

            void landmark(Canvas c, Size s) {
              final y = g.lmY;
              if (y < -300 || y > s.height + 300) return;
              final cx = g.lmKind.isEven ? g.roadL / 2 : s.width - g.roadL / 2;
              final bw = g.roadL * 0.8;
              final p = Paint();
              switch (g.lmKind) {
                case 0:
                  p.color = const Color(0xFFB5651D);
                  c.drawRect(
                      Rect.fromCenter(center: Offset(cx, y), width: bw * 0.7, height: 170),
                      p);
                  p.color = const Color(0xFF8A4A12);
                  for (int i = 0; i < 6; i++) {
                    c.drawRect(
                        Rect.fromLTWH(cx - bw * 0.25, y - 70 + i * 24, bw * 0.5, 6), p);
                  }
                  break;
                case 1:
                  p.color = const Color(0xFF2B6CB0);
                  c.drawRect(
                      Rect.fromCenter(center: Offset(cx, y), width: bw, height: 120), p);
                  p.color = Colors.white;
                  c.drawRect(
                      Rect.fromCenter(center: Offset(cx, y - 40), width: bw, height: 14),
                      p);
                  p.color = const Color(0xFF1A2B3C);
                  c.drawRRect(
                      RRect.fromRectAndRadius(
                          Rect.fromCenter(
                              center: Offset(cx, y + 20), width: bw * 0.35, height: 60),
                          const Radius.circular(20)),
                      p);
                  break;
                case 2:
                  p.color = Colors.white;
                  c.drawRect(
                      Rect.fromCenter(center: Offset(cx, y), width: bw, height: 120), p);
                  p.color = const Color(0xFF1B8A4B);
                  c.drawRect(
                      Rect.fromCenter(center: Offset(cx, y), width: bw * 0.7, height: 80),
                      p);
                  break;
                default:
                  p.color = const Color(0xFFC9A66B);
                  c.drawRect(
                      Rect.fromCenter(center: Offset(cx, y), width: bw, height: 110), p);
                  p.color = const Color(0xFF5A3F1E);
                  c.drawRRect(
                      RRect.fromRectAndRadius(
                          Rect.fromCenter(
                              center: Offset(cx, y + 15), width: bw * 0.4, height: 70),
                          const Radius.circular(20)),
                      p);
              }
            }

            void drawObj(Canvas c, Obj o) {
              final x = g.laneX(o.lane.toDouble());
              final w = g.objW(o.type), h = g.objH(o.type);
              final r = Rect.fromCenter(center: Offset(x, o.y), width: w, height: h);
              final p = Paint();
              switch (o.type) {
                case 0:
                  car(c, r, const Color(0xFF1E73BE));
                  break;
                case 1:
                  car(c, r, const Color(0xFFF2E6C9));
                  break;
                case 2:
                  car(c, r, const Color(0xFF2E8B57));
                  break;
                case 3:
                  p.color = const Color(0xFFFF7A00);
                  c.drawCircle(Offset(x, o.y), w / 2, p);
                  p.color = Colors.white;
                  c.drawCircle(Offset(x, o.y), w / 4, p);
                  break;
                default:
                  p.color = const Color(0xFFE0A030);
                  c.drawRRect(RRect.fromRectAndRadius(r, const Radius.circular(4)), p);
                  p.color = const Color(0xFFB07010);
                  c.drawRect(Rect.fromCenter(center: Offset(x, o.y), width: 5, height: h), p);
                  c.drawRect(Rect.fromCenter(center: Offset(x, o.y), width: w, height: 5), p);
              }
            }

            void car(Canvas c, Rect r, Color col) {
              final p = Paint()..color = col;
              c.drawRRect(RRect.fromRectAndRadius(r, const Radius.circular(8)), p);
              p.color = const Color(0xCC1B2530);
              c.drawRect(
                  Rect.fromLTWH(r.left + r.width * 0.12, r.top + r.height * 0.14,
                      r.width * 0.76, r.height * 0.14),
                  p);
              c.drawRect(
                  Rect.fromLTWH(r.left + r.width * 0.12, r.bottom - r.height * 0.26,
                      r.width * 0.76, r.height * 0.12),
                  p);
            }

            void truck(Canvas c) {
              if (g.inv > 0 && (g.inv * 10).floor().isEven) return;
              final lw = g.laneW;
              final tw = lw * 0.6, th = lw * 1.9;
              final x = g.laneX(g.lane), y = g.truckY;
              final top = y - th / 2;
              final p = Paint()..color = Colors.black;
              for (final dy in [0.1, 0.5, 0.85]) {
                c.drawRRect(
                    RRect.fromRectAndRadius(
                        Rect.fromLTWH(x - tw / 2 - 3, top + th * dy, 6, th * 0.1),
                        const Radius.circular(2)),
                    p);
                c.drawRRect(
                    RRect.fromRectAndRadius(
                        Rect.fromLTWH(x + tw / 2 - 3, top + th * dy, 6, th * 0.1),
                        const Radius.circular(2)),
                    p);
              }
              p.color = Colors.white;
              c.drawRRect(
                  RRect.fromRectAndRadius(
                      Rect.fromLTWH(x - tw / 2, top + th * 0.32, tw, th * 0.68),
                      const Radius.circular(6)),
                  p);
              p.color = const Color(0xFFC1272D);
              c.drawRect(Rect.fromLTWH(x - tw / 2, top + th * 0.6, tw, th * 0.08), p);
              c.drawRRect(
                  RRect.fromRectAndRadius(
                      Rect.fromLTWH(x - tw / 2 + 2, top, tw - 4, th * 0.3),
                      const Radius.circular(8)),
                  p);
              p.color = const Color(0xFF9EDCF5);
              c.drawRect(
                  Rect.fromLTWH(x - tw / 2 + 7, top + th * 0.06, tw - 14, th * 0.08), p);
            }
          }
          EOF
          flutter build apk --release
      - uses: actions/upload-artifact@v4
        with:
          name: rabat-truck-apk
          path: build/app/outputs/flutter-apk/app-release.apk
