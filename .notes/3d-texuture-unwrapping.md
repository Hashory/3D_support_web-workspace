# 3D テクスチャ展開 (自動 UV Unwrapping) 手法サーベイ

調査日: 2026-09-28
目的: 3D モデルのテクスチャ展開 (UV 展開 / パラメータ化 / アトラス生成) を自動化する代表的手法を網羅し、特徴・位置づけ・処理量・文献/実装を整理する。

> 用語: UV unwrapping = 3D 表面 → 2D 平面への写像。大きく (1) カット/チャート分割 (seam/chart segmentation) と (2) 平坦化 (flattening/parameterization) と (3) パッキング (packing) の3段階からなる。多くの「自動化」手法はこのいずれか、または joint 最適化である。

## 目次

- 0. 全体像・比較表
- A. 古典/基礎パラメータ化 (カット所与で1チャートを潰す)
  - A1. Harmonic / Tutte / Floater (線形・固定境界)
  - A2. LSCM
  - A3. ABF / ABF++
  - A4. ARAP パラメータ化
  - A5. MIPS / Harmonic 系・等長系エネルギー群
  - A6. BFF (Boundary First Flattening)
  - A7. SLIM / SCAF / AQP・KVF系 (反転なし大域最適化)
- B. チャート分割・シーム決定 (どこを切るか)
  - B1. Stretch-driven: Sander et al. / TOG Sander-Gortler-Snyder-Hoppe 系
  - B2. VSA (Variational Shape Approximation)
  - B3. D-Charts
  - B4. Iso-charts / Microsoft UVAtlas
  - B5. Seamster
  - B6. Multi-Chart Geometry Images (MCGIM)
  - B7. AutoCuts
  - B8. OptCuts
  - B9. Variational Surface Cutting (VSC) / SeamCut 系
- C. 実用自動ツール・ライブラリ
  - C1. xatlas (thekla_atlas フォーク)
  - C2. Blender 内蔵群 (Unwrap ABF/Conformal, Smart UV, Lightmap Pack, 投影系, Pack Islands)
  - C3. 商用・その他 (RizomUV / Maya / Ministry of Flat / Blender アドオン等)
  - C4. 汎用ジオメトリ処理基盤 (libigl / CGAL)
- D. UVレス代替 (展開自体を不要にする)
  - D1. Ptex
  - D2. Triplanar / Box / Procedural 投影
- E. 学習ベース新手法 (2023–2026)
  - E1. Nuvo
  - E2. FAM (Flatten Anything Model) / FlexPara
  - E3. GraphSeam
  - E4. SeamGPT (Auto-Regressive Surface Cutting)
  - E5. SeamCrafter (SeamGPT + DPO/RL)
  - E6. SeamGen (Graph Flow Matching)
  - E7. MeshTailor
  - E8. ArtUV (SeamGPT + AE リファイン)
  - E9. DreamUV (End-to-End Flow Matching)
  - E10. PartUV (参考)
- F. 手法選定ガイド
- G. 評価指標の補足
- 参考文献・リンク集

---

## 0. 全体像・比較表

| 手法 | 分類 | 自動度 | 歪みの種類 | 反転保証 | 処理速度の目安 | 向き |
|---|---|---|---|---|---|---|
| Harmonic/Tutte/Floater | 平坦化・線形 | △ (境界固定要) | 角度◎/面積× | 条件付き○ (正重みでOK) | 最速 (疎線形1回) | ベースライン、教材 |
| LSCM | 平坦化・線形 | ○ (自由境界) | 共形 (角度) | × (折返しあり得る) | 速い (疎線形1回、10万面で秒) | 有機物、Blender Conformal |
| ABF/ABF++ | 平坦化・非線形 | ○ (自由境界) | 共形、低ストレッチ | ○ (理論上反転なし) | 中 (ABFは遅い、ABF++は10倍速・階層版は数百万面を秒) | 高品質共形、Blender Angle Based |
| ARAP param | 平坦化・非線形 | ○ | 等長 (as-rigid-as-possible) | 初期値依存 | 中 (local-global反復、疎線形×数十回) | 等長重視、libigl定番 |
| BFF | 平坦化・線形 | ○ +境界制御 | 共形+面積最小 | △ | 速い (Laplace分解1回、従来比~50x、数百万面でも対話的) | 境界形状指定、対話編集 |
| SLIM/SCAF | 平坦化・非線形 | ○ | 共形/等長選択可 (Symmetric Dirichlet等) | ○ (flip-preventing) | 中〜重 (反復毎に疎線形、25M面で80分等の報告) | 反転ゼロ保証が要る高品質用途 |
| VSA | 分割 | ○ (プロキシ数指定) | 近似誤差 | — | 中 | 平面近似分割の祖、CAD/リメッシュ前処理 |
| D-Charts | 分割+平坦化 | ○ | developability指標で等長寄り | — | 中 | 準可展分割、L2 stretch小 |
| Iso-charts/UVAtlas | 分割+平坦化+パック | ◎ (全自動) | stretch最小 | — | 中〜重 (最適化+パック) | ゲーム/ライトマップ量産、DirectX系 |
| Seamster | シーム配置 | ○ | 低歪+目立たない配置 | — | 中 (MST/スケルトン系) | キャラ等、見た目配慮 |
| AutoCuts | joint (カット+歪み同時) | ◎ | 歪み-シーム数トレードオフ | △ | 対話的〜分単位 | 自動UVの近代的起点 |
| OptCuts | joint+双射保証 | ◎ (歪み上限のみ指定) | 歪み上限を厳密満足+シーム最短 | ○ (scaffoldで大域双射) | 重 (数百秒〜、AutoCutsより高速・省メモリ) | 歪み上限保証が必要な自動化 |
| xatlas | 分割+平坦化+パック | ◎ | 実用バランス (歪み小だが断片化多) | — | 速い (10万面で秒オーダー) | ベイク/ライトマップ/生成AI前処理のデファクトOSS |
| Blender Smart UV | 分割+投影 | ◎ | 角度閾値依存、歪み大の場合あり | — | 最速 | ハードサーフェス量産 |
| Ptex | UVレス | ◎ (展開不要) | なし (面毎テクスチャ) | — | 展開コスト0 (描画時フィルタ負荷) | 映画Subdiv、本レポートでは対照軸 |
| Nuvo | 学習・neural field | ◎ (不良メッシュ可) | conformal+stretch損失 | soft制約 | 重 (シーン毎最適化、GPUで数十分〜) | NeRF/生成メッシュ等“荒い”形状 |
| FAM | 学習・教師なし | ◎ (カット不要) | cycle+共形正則化 | soft制約 | 重 (per-shape最適化で数十分、SeamCrafter計測で~3000s級) | 点群OK、トポロジ不問 |
| SeamGPT系 | 学習・生成 (シームのみ) | ◎ (アーティスト風) | 後段展開に依存 | — | 学習重・推論軽〜中 (数十〜数百秒) | 人手風・意味的シームが欲しい場合 |
| ArtUV/DreamUV | 学習・生成 (シーム+UV end-to-end) | ◎ | 歪み+直線性+利用率 | soft/後処理 | 推論は数十秒級 (学習は大規模) | プロダクション風レイアウト再現 |

処理速度はメッシュ規模・ハード・実装に強く依存する。以下各節の「処理量」を参照。

---

## A. 古典/基礎パラメータ化

前提: 境界 (カット) が与えられたディスク位相パッチをどう潰すか。自動パイプラインの「平坦化エンジン」として今も現役。

### A1. Harmonic / Tutte / Floater Mean Value (固定境界・線形)

- 特徴: 境界頂点を凸形状 (円・正方形) に固定し、内部を調和写像 (Laplace 方程式) で解く。Tutte embedding (1963) が理論的祖。Floater の shape-preserving / mean value coordinates (1997/2003) が重み改良。正重みなら単射 (折返しなし) が保証されるのが最大の利点。
- 位置づけ: 最も単純・堅牢・高速なベースライン。反面、境界固定が歪みを生む (細長い突起で面積歪み大)。LSCM/ABF 等の「自由境界」手法に対する対照軸。libigl `harmonic`、CGAL `Tutte/MVC` が実装。
- 処理量: 疎線形方程式1回 (Cholesky等)。10万〜100万面でも秒オーダー。メモリも小。
- 論文/URL:
  - Tutte, "How to draw a graph", Proc. LMS 1963.
  - Floater, "Parametrization and smooth approximation of surface triangulations", CAGD 1997. PDF: https://www.cs.jhu.edu/~misha/Fall09/Floater97.pdf
  - Floater, "Mean value coordinates", CAGD 2003.
  - サーベイ: Floater & Hormann, "Surface Parameterization: a Tutorial and Survey", https://graphics.stanford.edu/courses/cs468-05-fall/Papers/param-survey.pdf ; Sheffer et al., "Mesh Parameterization Methods and Their Applications", https://www.cs.ubc.ca/~sheffa/papers/param_survey06.pdf
- 実装:
  - libigl tutorial 501 Harmonic: https://libigl.github.io/tutorial (Chapter 5)
  - CGAL Surface_mesh_parameterization (Tutte / MVC / Discrete Authalic/Conformal / ARAP / LSCM / Orbifold): https://doc.cgal.org/latest/Surface_mesh_parameterization/index.html

### A2. LSCM (Least Squares Conformal Maps)

- 特徴: Cauchy-Riemann 方程式の最小二乗近似で共形性を最適化 (Lévy et al. 2002)。自由境界のため固定境界法より角度歪みが小さい。線形最小二乗に帰着し、2頂点固定で一意解。大きなチャート・複雑境界に強い。同時にチャート成長+パッキングまで含めた「自動テクスチャアトラス生成」の古典的完成形。
- 位置づけ: 共形系のデファクト。Blender の "Conformal" は LSCM。ABF より速いが ABF より歪みはやや大きく、折返し (flip) が起き得る。BFF の論文が「drop-in replacement対象」と名指しする基準手法。
- 処理量: 疎線形1回。Blender 実装メモによれば SuperLU 等で高速。10万面級で秒オーダー。チャート分割・パッキング込みでも軽い部類。
- 論文: Lévy, Petitjean, Ray, Maillot, "Least Squares Conformal Maps for Automatic Texture Atlas Generation", SIGGRAPH 2002. PDF: https://www.cs.jhu.edu/~misha/Fall09/Levy02.pdf
- 実装:
  - Blender `parametrizer.c` (LSCM/ABF): https://archive.blender.org/wiki/2015/index.php/Dev:Source/Blender/Textures/UV/Unwrapping
  - C++/Eigen 例: https://github.com/kyle-gh/LeastSquaresConformalMaps , https://github.com/math-castro/lscm
  - Python/教材: https://perso.liris.cnrs.fr/vincent.nivoliers/cgdi/documents/notebooks/lscm.html
  - LSCM+シーム+パック例: https://github.com/MukeshK17/uv-unwrapping
  - libigl 502 / CGAL LSCM もあり (上記リンク)。

### A3. ABF / ABF++ (Angle Based Flattening)

- 特徴: 角度空間で直接最適化する共形手法 (Sheffer & de Sturler 2001 → Sheffer et al. 2005)。制約 (内点角度和=2π、三角形内角和=π、辺長整合) 下で 3D 角→2D 角を非線形最適化し、LSCM で座標復元。provably valid (反転三角形なし)・低ストレッチ。
- 位置づけ: LSCM より高品質・低歪みだが本来は遅い。Blender の "Angle Based" は ABF 系で、LSCM より良い 2D 表現だが複雑形状では重い。DreamUV 論文の比較でも ABF/ABF++ は単一チャートで滑らかだが境界が非直線・回転/剪断が出ると報告。
- 処理量: オリジナル ABF は反復毎に大規模線形系 (60K面で5分との報告)。ABF++ は Hessian 次元を1/5に削減し最大10倍高速、数十万面を数分。階層 (hierarchical) 版は数百万面を秒で解くが歪み微増。Blender マニュアルも Angle Based > Conformal の計算負荷を示唆。
- 論文: Sheffer, Lévy, Mogilnitsky, Bogomyakov, "ABF++: Fast and Robust Angle Based Flattening", ACM TOG 2005. PDF: https://www.cs.ubc.ca/~sheffa/papers/abf_plus_plus.pdf / ACM: https://dl.acm.org/doi/10.1145/1061347.1061354
- 実装: Blender 内蔵 (parametrizer)、UBC 紹介頁 https://www.cs.ubc.ca/~sheffa/ABF++/abf.htm 。単体OSSは少ない (Blender/CGAL/libigl 経由利用が現実的)。

### A4. ARAP パラメータ化 (As-Rigid-As-Possible)

- 特徴: 「局所をなるべく剛体に」をエネルギー化 (Sorkine & Alexa 2007 の変形理論を Liu et al. 2008 がパラメータ化へ拡張)。local (各三角の最適回転を SVD) / global ( stitching の線形 solve) を交互反復。等長性 (isometric) が良く、チェッカー歪みが少ない。境界条件の与え方で結果が変わる。
- 位置づけ: 共形 (LSCM/ABF) 対 等長 (ARAP) の対照軸。等長・剛性保持が欲しい布/リメッシュ/変形連動に向く。SLIM の local-global 反復の原型でもある。
- 処理量: 反復毎に疎線形1回 + 全三角形 SVD。数十反復で収束することが多く、中規模は秒〜分。初期値 (harmonic 等) 依存で局所解に落ちる場合あり。
- 論文:
  - Sorkine & Alexa, "As-Rigid-As-Possible Surface Modeling", SGP 2007. https://igl.ethz.ch/projects/ARAP/
  - Liu, Zhang, Su, Bao, "Reconstruction of ... / A Local/Global Approach to Mesh Parameterization" (ARAP param, 2008).
- 実装: libigl tutorial 503 (`arap_precomputation`/`arap_solve`, dim=2 でパラメータ化): https://github.com/libigl/libigl/blob/main/tutorial/503_ARAPParam/main.cpp ; CGAL ARAP parameterizer (上記 CGAL 頁)。

### A5. MIPS / 等長・共形エネルギー群 ( stretch, symmetric Dirichlet 等)

- 特徴: Hormann & Greiner の MIPS (Most Isometric Parameterizations, 2000) を祖に、Sander et al. の geometric stretch (L2/L∞)、Smith & Schaefer の symmetric Dirichlet などで「何を測って最小化するか」を定式化。面積保存 vs 角度保存のトレードオフをエネルギーが直接表現する。後の SLIM/OptCuts の歪み上限・目的関数 (例: symmetric Dirichlet <  bound) の語彙になっている。
- 位置づけ: 単体手法というより評価・目的関数の標準。論文間比較は大抵この指標で揃える (SeamCrafter/DreamUV/MeshTailor の表も distortion = 面積/角度系)。
- 処理量: エネルギー自体は局所計算で安い。最適化は非線形 (Newton/quasi-Newton) で BFF/SLIM/AQP 等の解法に委ねられる。
- 文献:
  - Hormann & Greiner, "MIPS: An Efficient Global Parametrization Method", 2000.
  - Sander, Snyder, Gortler, Hoppe, "Texture Mapping Progressive Meshes", SIGGRAPH 2001. https://dash.harvard.edu/entities/publication/73120378-7cc6-6bd4-e053-0100007fdf3b
  - Smith & Schaefer, "Bijective Parameterization with Free Boundaries", SIGGRAPH 2015 (symmetric Dirichlet).
  - Floater & Hormann survey (上記)。

### A6. BFF (Boundary First Flattening)

- 特徴: 共形平坦化を「境界→内部」の順に線形で解く (Sawhney & Crane, TOG 2017)。Cherrier 公式 + Poincaré–Steklov 作用素で境界データを求め、内部は調和拡張。Laplace 行列の分解1回で済み、境界長/角度の直接制御・角保持・cone 特異点・円盤一様化・任意目標形状への写像まで単一枠組みで扱える。
- 位置づけ: 「LSCM の drop-in replacement」かつ「従来の境界制御付き共形法より50倍安い」。対話的 UV 編集・高速最適化の内側ループに向く。単一チャート用なので自動カットとは直交 (組み合わせて使う)。
- 処理量: 論文主張: 実 Laplace 行列の単一 factorization のみ、償却コストは既存境界制御法の約1/50。数百万三角形でも対話編集可能。アプリ同梱。
- 論文: Sawhney & Crane, "Boundary First Flattening", ACM TOG 37(1), 2017. arXiv: https://arxiv.org/abs/1704.06873 , ACM: https://dl.acm.org/doi/10.1145/3132705
- 実装: 公式アプリ+コード https://github.com/GeometryCollective/boundary-first-flattening (Windows v1.6 バイナリあり)。

### A7. SLIM / SCAF / AQP・KVF 系 (反転なし大規模最適化)

- 特徴: flip-preventing エネルギー (symmetric Dirichlet 等、反転で発散) を、ARAP 的 local-global の proxy 列で間接最小化 (Rabinovich et al., TOG 2017)。初期値が高歪みでも反転なしを保ちつつ低歪みに収束。2D/3D 双方・変形/品質改善にも適用可。派生に SCAF (Scalable Conformal?/Seamless おむね同系統)、AQP (Accelerated Quadratic Proxy)、KVF preconditioner (Solomon et al., "Isometry-Aware Preconditioning") がある。
- 位置づけ: 「品質は非線形最適化並み、規模は数百万〜2500万面」を狙う実戦的高品質路線。libigl 709 SLIM / 710 SCAF として叩ける。FAM 論文の比較対象 (SLIM) でもある。
- 処理量: 反復毎に疎線形1回 + 三角毎 2x2 SVD。例: 2500万三角形を40反復・80分 (ETH頁)。通常規模 (数万〜数十万面) なら秒〜分。Newton 法より大規模で有利との報告 (KVF 論文の 550K/8.8M 比較)。
- 論文: Rabinovich, Poranne, Panozzo, Sorkine-Hornung, "Scalable Locally Injective Mappings", ACM TOG 2017. https://igl.ethz.ch/projects/slim (+ tech report PDF https://igl.ethz.ch/projects/slim/slim-techreport-2016.pdf , ACM https://dl.acm.org/doi/10.1145/3072959.2983621 )
- 実装: 最適化並列コード (ETH頁の GitHub)、libigl 参照実装・tutorial 709/710、比較論文 https://people.csail.mit.edu/jsolomon/assets/killing_param.pdf 。

---

## B. チャート分割・シーム決定

### B1. Stretch-driven 分割 (Sander–Snyder–Gortler–Hoppe 系)

- 特徴: テクスチャ stretch (L2/L∞) を最小化するチャート成長 + チャート毎パラメータ化 + stretch 比例リサイズ + アトラスパック。PM (progressive mesh) 全系列で共通パラメータ化を保つ (2001)。平面性・コンパクト性ヒューリスティクスで貪欲クラスタリング。
- 位置づけ: ゲーム的「歪み均一・mip に強い」アトラスの祖。Iso-charts/UVAtlas、xatlas の chart merging コストの源流。
- 処理量: 貪欲成長+反復最適化で中規模は秒〜分。2001年当時で実用速度。
- 論文: Sander, Snyder, Gortler, Hoppe, "Texture Mapping Progressive Meshes", SIGGRAPH 2001 (上記リンク)。

### B2. VSA (Variational Shape Approximation)

- 特徴: 形状近似を「幾何プロキシへの変分的分割」として定式化 (Cohen-Steiner, Alliez, Desbrun, SIGGRAPH 2004)。Lloyd 反復で面クラスタ→最適平面プロキシを交互改善。平面プロキシ分割はそのまま準可展チャート分割の前処理になる。
- 位置づけ: 分割の汎用理論。D-Charts/Iso-charts の下敷き。CGAL に公式実装あり。
- 処理量: クラスタ数・反復依存。平面フィットは安いが大規模は反復が支配的。
- 論文: "Variational Shape Approximation", ACM TOG 2004. PDF: https://www.geometry.caltech.edu/pubs/CAD04.pdf , ACM: https://dl.acm.org/doi/10.1145/1015706.1015817
- 実装: CGAL Triangulated Surface Mesh Approximation: https://doc.cgal.org/latest/Surface_mesh_approximation/index.html (BubCGAL blog https://www.cgal.org/2019/01/29/VSA/ )。

### B3. D-Charts (Quasi-Developable Segmentation)

- 特徴: 可展性 (developability) 指標を新規定義し、分割段階でアトラス歪みをバウンド (Julius, Kraevoy, Sheffer, Eurographics 2005)。同チャート数で従来より等長的なアトラス。Fandisk 等の機械物で L2 stretch ≈1 と報告。
- 位置づけ: VSA→実用アトラスの橋渡し。Iso-charts との比較対象の定番 (Sheffer survey の Fig.11)。
- 処理量: 中規模で秒〜分オーダー (指標計算+成長)。
- 論文: Julius, Kraevoy, Sheffer, "D-Charts: Quasi-Developable Mesh Segmentation", CGF 2005. PDF: https://www.cs.ubc.ca/labs/imager/tr/2005/Vlad_DCharts/EG05.pdf , 紹介 https://www.cs.ubc.ca/~vlady/dcharts/dcharts.htm

### B4. Iso-charts / Microsoft UVAtlas

- 特徴: スペクトル解析で特徴→iso-contour 分割→各特徴を形状分類 (平面/円筒/球: 長手・円形・野球縫いカット) する全自動 stretch-driven 法 (Zhou et al., SGP 2004)。Microsoft が UVAtlas (isochart texture atlasing) として製品化。`UVAtlasCreate/Partition/Pack` + IMT 計算まで含む。
- 位置づけ: ゲーム/ベイク量産の実務標準の1つ。xatlas と双璧だが、UVAtlas は DirectX 系・現在は archived (レガシー推奨)。品質は安定だが過分割気味との指摘もある。
- 処理量: スペクトル分解+最適化のため中〜重。コマンドラインツールで計測可。
- 論文: Zhou, Snyder, Guo, Shum, "Iso-charts: Stretch-driven Mesh Parameterization using Spectral Analysis", SGP 2004.
- 実装: https://github.com/microsoft/UVAtlas (★900超、MIT、wiki あり。archived 表記に注意)。

### B5. Seamster (Inconspicuous Low-Distortion Seam Layout)

- 特徴: 「歪みが小さく目立たない」シームを MST/領域成長で配置 (Sheffer & Hart, IEEE Vis 2002)。ABF++ 論文の図も Seamster カットを使用。可視性 (visibility) と歪みを両立。
- 位置づけ: キャラ等の見た目重視シームの古典。後の GraphSeam の後処理 (Steiner tree + skeletonization) の源流として引用される。
- 処理量: グラフ最適化主体で軽〜中。
- 論文: Sheffer & Hart, "Seamster: Inconspicuous Low-Distortion Texture Seam Layout", IEEE Visualization 2002.

### B6. MCGIM (Multi-Chart Geometry Images)

- 特徴: 任意表面を規則 2D グリッドに再サンプル (Sander et al. 2003)。チャート化→L2 stretch 最小パラメータ化→相対スケール→パック。GPU フレンドリ。
- 位置づけ: ベイク・GPU 処理との相性が良い表現。xatlas/UVAtlas とは出力表現 (regular grid) が異なる軸。
- 論文: Sander, Wood, Gortler, Snyder, Hoppe, "Multi-Chart Geometry Images", SGP 2003. PDF: https://www.multires.caltech.edu/pubs/mcgim_final.pdf

### B7. AutoCuts (同時歪み・カット最適化)

- 特徴: カットと歪みを同一エネルギーで同時最適化した初期の joint 法 (Poranne et al., SIGGRAPH 2017)。シーム数と歪みをバランスし、対話レートで動作。OptCuts の直接の前任。
- 位置づけ: 「切ってから潰す」2段階からの脱却点。OptCuts との比較で「長時間・メモリ大」の旧世代として言及されることもあるが、自動化の起点として重要。
- 処理量: 対話的〜分単位を謳う。OptCuts スライドのベンチでは AutoCuts 455s に対し OptCuts 374s 等 (条件依存)。
- 論文: Poranne, Tarini, Huber, Panozzo, Sorkine-Hornung, "AutoCuts: Simultaneous Distortion and Cut Optimization for UV Mapping", ACM TOG 2017. PDF: https://cims.nyu.edu/gcl/papers/Autocuts-2017.pdf

### B8. OptCuts (Joint Optimization of Surface Cuts and Parameterization)

- 特徴: 任意初期埋め込み+ユーザ指定の歪み上限から、トポロジー更新 (cut locus の split/merge) とジオメトリ更新を交互に単調減少させ、歪み上限を厳密に満たす最短シーム・大域双射 UV を自動生成 (Li et al., SIGGRAPH Asia 2018)。scaffold 三角形で大域双射維持。領域指定シーム・既存埋め込み改善にも対応。パラメータ調整不要。
- 位置づけ: 最適化系自動 UV の現時点での到達点の1つ。Nuvo/MeshTailor/SeamCrafter 等の学習系論文が一律で比較対象にする基準線。xatlas よりチャート少なく短シームだが島形状は不規則になり得る。
- 処理量: 重い部類。71形状ベンチで数百秒、スケーラビリティ図では頂点増で AutoCuts が OOM する領域でも動作。最新版で平均3倍高速化との記載あり。Nuvo 比較では bunny/lion で競合、Zip-NeRF メッシュでは失敗する場合あり。
- 論文: Li, Kaufman, Kim, Solomon, Sheffer, "OptCuts: Joint Optimization of Surface Cuts and Parameterization", ACM TOG 37(6), 2018. 紹介 https://research.adobe.com/publication/optcuts-joint-optimization-of-surface-cuts-and-parameterization , プロジェクト (PDF/Video/Code/Data) https://www.cs.ubc.ca/labs/imager/tr/2018/OptCuts , ACM https://dl.acm.org/doi/10.1145/3272127.3275042 , スライド https://slides.games-cn.org/pdf/GAMES201879%E6%9D%8E%E6%97%BB%E8%BE%B0.pdf
- 実装: 公式 GitHub (プロジェクト頁の Source Code and Data)、非公式ミラー https://github.com/hongbin-y/OptCuts_NewUVUnwrap , https://github.com/Liz490/OptCuts-uv-mapping 。

### B9. VSC / SeamCut 系 (場ベース切断)

- 特徴: Sharp & Crane 2018 の Variational Surface Cutting、Lucquin et al. の SeamCut 等、ベクトル場・スカラー場の最適化で切断を求める数学的にエレガントな系統。SeamCrafter 論文の整理では「初期値・パラメータに敏感で実務堅牢性に課題」と位置づけ。
- 位置づけ: 最適化系と学習系の中間に置かれる参照軸。実装は研究コード止まりが多い。
- 論文: Sharp & Crane, "Variational Surface Cutting", ACM TOG 2018.

---

## C. 実用自動ツール・ライブラリ

### C1. xatlas

- 特徴: Jonathan Young による C++11・依存なし・MIT のメッシュパラメータ化/UV ライブラリ。thekla_atlas (The Witness 実績) の独立フォーク。`Create → AddMesh → Generate` でチャート分割→パラメータ化→パックまで一括。ライトマップ/ベイク向け unique UV 生成のデファクト OSS。glTF-Transform 連携例、WASM 版もある。
- 位置づけ: 研究的高品質 (OptCuts) と実務速度のトレードオフで「速く・壊れず・過分割許容」側。学習系論文のベースラインとして毎回登場し、「歪み最小だが断片化が最大」「Nuvo より編集性低」の定型評価を受ける。SeamGPT/SeamCrafter 計測では Toys4K 等で distortion 1.86〜35、fragments 数十〜数百、runtime 秒〜数十秒。
- 処理量: 速い。10万面級で秒オーダー、実務で量産可。品質設定で変動。
- 実装: https://github.com/jpcy/xatlas (★2.5k)。使い方例 https://github.com/jpcy/xatlas/blob/master/source/examples/example.cpp 。glTF 連携議論 https://github.com/donmccurdy/glTF-Transform/discussions/792 。

### C2. Blender 内蔵群

Blender は研究手法 (ABF/LSCM) と実務ヒューリスティクスを同居させた最大の実用例。すべて `U` → Unwrap 系 + UV Editor の Pack Islands で完結する。

| 機能 | 特徴 | 自動度/速度 |
|---|---|---|
| Unwrap Angle Based (ABF) | 良好な2D表現、複雑有機物向き | シーム要、遅め |
| Unwrap Conformal (LSCM) | 単純物向き、Angle Basedより不正確な場合あり | シーム要、速め |
| Smart UV Project | 面間角度閾値で自動切断→平均法線投影。機械/建築向き | 全自動・最速級 |
| Lightmap Pack | 面毎に分離配置。ベイク最大利用、絵付けには不向き | 全自動・高速、Pack Qualityで時間増 |
| Cube/Cylinder/Sphere Projection | 定型投影。重なりは Pack Islands で解消 | 全自動・最速 |
| Follow Active Quads | アクティブ四角起点で再帰貼付。トポロジ依存 | 半自動 |
| Pack Islands / UV Unwrap ノード | 回転・余白考慮パック。Geometry Nodes でも呼出可 | 自動 |

- 位置づけ: 「研究法 (ABF/LSCM) はシーム前提」「量産は Smart UV/投影+パック」の二層構造。Min Stretch (旧称 Minimum Stretch) も同系統。商用 (Maya/Rizom) との機能対応表の基準になる。
- 処理量: すべて対話的。Smart UV/投影は一瞬、ABF のみ大規模で待つ。
- URL: UV Operators (5.2 LTS) https://docs.blender.org/manual/en/latest/modeling/meshes/editing/uv.html , Seams https://docs.blender.org/manual/en/latest/modeling/meshes/uv/unwrapping/seams.html , UV Unwrap ノード https://docs.blender.org/manual/en/latest/modeling/geometry_nodes/mesh/uv/uv_unwrap.html 。

### C3. 商用・その他ツール (RizomUV / Autodesk Maya / Ministry of Flat 等)

- 特徴: RizomUV (Unfold3D 系譜) はストレッチ表示・対称・パックに強いアーティスト標準。Maya の Automatic/Planar/Cylindrical + Unfold、3ds Max の Peel/Pack 同等。Ministry of Flat は商用自動アンラッパーとして xatlas README に列挙。Headus UVLayout も同様の対話自動。
- 位置づけ: 本レポートの最適化・学習手法の「人手上限/代替」の参照点。ArtUV 論文は Blender/Maya/3dsMax との定性・定量比較 (distortion/overlap/utilization/artist-level) を行っている。
- 処理量: 対話的。ライセンス・UI 習熟が支配的。
- URL: 各公式 (RizomUV, Autodesk Maya UV)。xatlas README の関連列挙も参照。

### C4. 汎用基盤 libigl / CGAL (+ CoMiSo)

- 特徴: 研究実装の共通語彙。libigl は header-only + Eigen で `harmonic/lscm/arap/SLIM/SCAF` を tutorial 501/502/503/709/710 で一気通貫。CGAL は商用利用可の堅牢実装 (LSCM/ARAP/Tutte/MVC/Authalic/Orbifold)。CoMiSo (mixed-integer) は seamless/global param (505) に必要で GPL3 注意。
- 位置づけ: 新手法のプロトタイピング・ベンチの土台。論文の「実装あり」は大抵このいずれか。
- 処理量: 手法本体に準ずる (A章参照)。
- URL: libigl tutorial https://libigl.github.io/tutorial , 例 https://github.com/libigl/libigl/tree/main/tutorial (501/502/503/709 SLIM/710 SCAF), `arap.h` https://github.com/libigl/libigl/blob/main/include/igl/arap.h ; CGAL https://doc.cgal.org/latest/Surface_mesh_parameterization/index.html 。

---

## D. UVレス代替

### D1. Ptex (Per-Face Texture Mapping)

- 特徴: Walt Disney Animation Studios (Burley & Lacewell, EGSR 2008)。Catmull-Clark/Loop/ポリメッシュの面毎に独立テクスチャ+隣接マップを単一ファイルに格納し、継ぎ目なし異方性フィルタで描画。UV 割当作業自体を消去。面毎に解像度可変 (64〜2048等)、数億 texel を単一 Ptex に格納した T-Rex 例が有名。2008年以降の Disney 作品のほぼ全表面で使用。
- 位置づけ: 「展開の自動化」ではなく「展開の消去」。Subdiv 前提・対応レンダラ/ペイント環境が要るためゲーム・汎用DCC とは非互換。対照軸として重要。
- 処理量: 展開コスト0。代わりにファイルI/O・キャッシュ・フィルタ実装が必要。
- 論文: "Ptex: Per-Face Texture Mapping for Production Rendering", EGSR 2008. https://ptex.us/ptexpaper.html , preprint https://media.disneyanimation.com/technology/opensource/ptex/ptex.pdf , ACM https://dl.acm.org/doi/10.1111/j.1467-8659.2008.01253.x
- 実装: https://github.com/wdas/ptex (★700超), 公式 https://disneyanimation.com/open-source/ptex , https://ptex.us/ 。

### D2. Triplanar / Box / Procedural 投影

- 特徴: 3軸 (あるいは球/円筒/立方) 投影をブレンドし UV なしで貼る。ワールド/オブジェクト空間でスケール・回転・オフセット指定。手続きテクスチャ・スカルプト試作・フォトグラメトリ仮貼りに多用。Blender の Tri Planar/UV Projection ノード、ゲームエンジンの box projection が該当。継ぎ目の完全消去は不可だが作業ゼロ。
- 位置づけ: 量産前の仮貼り・地形/岩等のタイリング用途。最終アセットの unique UV (xatlas/OptCuts) とは役割分担。
- 処理量: リアルタイム (シェーダ負荷のみ)。
- 参考: Blender Redshift UV Projection https://help.maxon.net/r3d/blender/en-us/Content/html/UV+Projection.html , JCGT cube-to-sphere https://www.jcgt.org/published/0007/02/01/paper-lowres.pdf 。

---

## E. 学習ベース新手法 (2023–2026)

共通前提: 「歪み最小」だけでは人手風 (直線・軸平行・意味的まとまり・編集容易性) にならない、という問題意識。ベンチは Toys4K/ShapeNet/FAM-benchmark/AIGC-100/ArtUV-200K 等、指標は distortion/fragmentation/runtime (+ boundary straightness/packing utilization/overlap/artist-level)。

### E1. Nuvo (Neural UV Mapping for Unruly 3D Representations)

- 特徴: メッシュ頂点に直接写像を持たず、neural field (chart割当MLP + 3D→UV + UV→3D の 2n+1 MLP) で「可視点のみ」正しく振る舞う連続 UV を最適化 (Srinivasan et al., ECCV 2024)。NeRF density 直接・DreamFusion 生成物・Zip-NeRF 抽出メッシュ等の不良形状に強い。xatlas が粉砕される例で編集可能アトラスを出す。
- 位置づけ: per-shape 最適化学習の先駆。OptCuts と良形 (bunny/lion) では競合、難形で優位。ただし全点双射の厳密保証はなく、ユーザ指定カット等の対話性なし。
- 処理量: 重い。シーン毎最適化で GPU 数十分〜 (SeamCrafter 計測: Toys4K 2569s、ShapeNet 2352s 等)。推論即時ではない。
- 論文: arXiv https://arxiv.org/abs/2312.05283 , ECCV https://www.ecva.net/papers/eccv_2024/papers_ECCV/papers/05571.pdf , 頁 https://pratulsrinivasan.github.io/nuvo
- 実装: 公式コード配布なし (非公式再実装 https://github.com/ruiqixu37/Nuvo )。

### E2. FAM (Flatten Anything Model) / FlexPara

- 特徴: Cut-Net (切断) + Deform-Net (UV変形) + Unwrap-Net (3D→2D) + Wrap-Net (2D→3D) の幾何解釈可能サブネットを双方向 cycle (2D→3D→2D / 3D→2D→3D) で教師なし最適化 (Zhang et al., NeurIPS 2024)。連結性を使わず表面点群で学習するため、メッシュ品質不問・点群・高 genus/実スキャン・AIGC に適用可。事前カット不要で切断線とUV境界を同時獲得。拡張 FlexPara で multi-chart 局所パラメータ化。
- 位置づけ: Nuvo と並ぶ per-shape 教師なし双璧。SLIM とのメッシュ比較、FAM-benchmark での後続比較の基準。SeamCrafter 評では「接続性非考慮で severe distortion の場合あり」との指摘もある。
- 処理量: 重い。per-model overfitting で GPU 数十分〜時間 (SeamCrafter 計測: Toys4K 3090s、FAM-benchmark 5656s)。点群対応の代償。
- 論文: NeurIPS https://proceedings.neurips.cc/paper_files/paper/2024/file/054f771d614df12fe8def8ecdbe4e8e1-Paper-Conference.pdf , arXiv https://arxiv.org/abs/2405.14633 , スライド https://neurips.cc/media/neurips-2024/Slides/94277.pdf
- 実装: https://github.com/keeganhk/FlattenAnything (★55)。

### E3. GraphSeam (Supervised Graph Learning for Semantic Seams)

- 特徴: GAT ベースの辺ラベル (seam/non-seam) 検出 + Steiner tree/skeletonization 後処理で意味的シームへ (Teimury et al., arXiv 2020)。AutoCuts/OptCuts を後段リファインに使うと歪み大幅減と報告。Autodesk Research の GraphSeam paper PDF あり。
- 位置づけ: 教師ありシーム検出の初期例。SeamGPT 系の「意味的シーム」路線の前任。
- 処理量: 学習後は GNN 推論+後処理で軽〜中。
- 論文/実装: https://www.research.autodesk.com/app/uploads/2024/08/GraphSeam-paper.pdf (preprint arXiv:2011.13748 系)。

### E4. SeamGPT (Auto-Regressive Surface Cutting)

- 特徴: 表面切断を next-token prediction 化。点群エンコーダで形状条件→GPT/hourglass transformer が量子化 3D 座標の線分列を軸ソート順に自己回帰生成 (Li et al., Tencent Hunyuan, arXiv 2025)。56万アーティストカットメッシュで学習。多様体/非多様体・スキャン・アーティスト物で SOTA、部位分割の clean boundary にも転用。ArtUV/SeamCrafter の土台。
- 位置づけ: 生成系シームの起点。xatlas/FAM/Nuvo との比較で distortion/fragmentation 両立 (SeamCrafter 表: Toys4K distortion 1.85/fragments 7.86/runtime 446s)。
- 処理量: 学習は大規模、推論は自己回帰のため中 (数百秒級の報告あり、メッシュ規模依存)。
- 論文: https://arxiv.org/abs/2506.18017 , PDF https://arxiv.org/pdf/2506.18017 , 頁 https://victorcheung12.github.io/seamgpt , OpenReview (ICLR2026) https://openreview.net/forum?id=9HeKCYl1zl
- 実装: 公式学習済み配布は要確認。非公式再実装 https://github.com/rahul237/seamgpt-iter 。

### E5. SeamCrafter (SeamGPT + DPO/RL)

- 特徴: SeamGPT 系の二分岐点群エンコーダ (幾何+位相) 事前学習→DPO 選好微調整 (歪み+断片化の seam 評価枠組みで pairwise labels)。過分割/高歪みシームを抑制し位相一貫性を保つ (Xu et al., arXiv 2025-09)。
- 位置づけ: 生成シームの品質整形版。Toys4K/ShapeNet/FAM-benchmark/AIGC-100 で xatlas/Nuvo/FAM/SeamGPT を上回ると報告 (例: Toys4K distortion 1.39/fragments 7.28/50s)。
- 処理量: 推論は SeamGPT より高速化 (表で 50s vs 446s 等、条件依存)。学習は重い。
- 論文: https://arxiv.org/abs/2509.20725 , HTML https://arxiv.org/html/2509.20725v1 。

### E6. SeamGen (Graph Flow Matching)

- 特徴: メッシュ辺グラフ上の per-edge seam 生成を flow matching で学習 (Transformer ではなく mesh-native Mesh Transformer)。ノイズ→アーティストマスクの条件付き transport。学習済み flow を training-free の inpainting prior として制約付き補完・重なり自己改善に再利用。Objaverse/Sketchfab の実 UV から学習 (8xA100, 200epoch)。SeamGPT/MeshTailor/SeamCrafter との対比で「全レイアウト同時生成・メッシュ位相考慮」を謳う。
- 位置づけ: 自己回帰 (SeamGPT) 対 一括生成 (flow) の対照軸。30K面超で GPU メモリ課題と明記。
- 処理量: 学習重、推論は flow sampling で中。
- 論文: https://arxiv.org/html/2607.12379v1 (arXiv:2607.12379)。

### E7. MeshTailor (Generative Mesh Traversal)

- 特徴: シームを生成的メッシュ走査で生成し、衣服整合・chain/loop の整った seam→規則的コンパクトチャートを狙う (Ma et al., arXiv 2026-03)。xatlas/Blender Smart/OptCuts/Nuvo/PartUV と比較し、歪み競合+チャート規則性で優位と報告。
- 位置づけ: アパレル/ハードサーフェスの「編集可能境界」重視路線。
- 処理量: 生成系に準ずる (中)。
- 論文: https://arxiv.org/pdf/2603.27309 (arXiv:2603.27309)。

### E8. ArtUV (Artist-style UV Unwrapping, SeamGPT + AE)

- 特徴: top-down 2段階。SeamGPT で意味的シーム→MiniStretch 等の粗 UV を Auto-Encoder で整形 (neatness/overlap/distortion 制約) しアーティスト風に (Chen et al., arXiv 2025-09, ICLR2026)。ArtUV-200K データセット (20万アーティスト UV) を整備・処理スクリプト公開予定。Blender/Maya/3dsMax との比較で neat/overlap/utilization/artist-level 優位。
- 位置づけ: 生成シームと最適化平坦化のハイブリッドの完成形の1つ。プラグイン/単体両用を謳う。
- 処理量: 推論は粗 UV + AE リファインで軽〜中。
- 論文: https://arxiv.org/abs/2509.20710 , OpenReview https://openreview.net/forum?id=LN7VQ3ed1t 。

### E9. DreamUV (End-to-End Flow Matching)

- 特徴: UV を単一最適解でなく「ノイズ→アーティスト UV 分布」の mesh-conditioned transport として flow matching 生成 (Ruan et al., arXiv 2026-06)。境界重視学習 + Model-in-the-Loop Finetuning (離散化誤差考慮) で直線境界・軸平行・タイトパックを再現。ABF/ABF++/xatlas/FAM と同一シーム比較で境界直線性 (S_rect)・packing (η_pack) 優位、歪み競合。プロアーティストのユーザスタディあり。
- 位置づけ: ArtUV (2段階) 対 DreamUV (end-to-end 生成) の対照軸。SeamGen (シーム生成) とも相補。
- 処理量: 学習大規模、推論は sampling で中。
- 論文: https://arxiv.org/abs/2606.22445 。

### E10. PartUV (参考, SIGGRAPH Asia 2025)

- 特徴: part priors でチャート数削減 (Wang et al.)。MeshTailor/SeamGen の比較対象。部位対応 UV が欲しい場合の選択肢。
- 論文: SIGGRAPH Asia 2025 Conference Papers (PartUV)。

---

## F. 手法選定ガイド

- とにかく速く量産 (ゲーム背景・小物・AIGC仮貼り): Blender Smart UV / Cube 投影 + Pack Islands、または xatlas。秒で unique UV。
- ベイク品質・ライトマップ (重なり禁止・利用率): xatlas / UVAtlas / Blender Lightmap Pack。xatlas が OSS 標準。
- 有機キャラで歪み最小・人手シームあり: Blender Angle Based (ABF) or LSCM + 手動シーム。反転ゼロ必須なら SLIM/SCAF。
- 歪み上限を厳密に守りたい自動化 (研究・高品質): OptCuts (歪み上限指定)。時間は許容する。
- 境界形状を指定・対話編集: BFF。
- Subdiv 映画・UV作業消去: Ptex (対応パイプライン要)。
- NeRF/生成・スキャンの荒い形状: Nuvo / FAM (per-shape 最適化、時間は重い)。
- 人手風・意味的・編集容易な UV (プロダクション): SeamGPT/SeamCrafter/SeamGen でシーム生成→ (ArtUV/DreamUV で整形) → DCC で微修正。現状の推奨ハイブリッド。
- ライブラリ開発: libigl (研究) / CGAL (堅牢) / xatlas (量産) の使い分け。Seamless が要るなら CoMiSo (GPL注意)。

---

## G. 評価指標の補足

- Conformal (角度) vs Isometric (等長・面積+角度): LSCM/ABF/BFF は前者、ARAP/SLIM (等長設定) は後者。用途で選択。
- Stretch L2/L∞ (Sander): テクスチャ sampling 均一性。値が小さいほど mip に強い。
- Symmetric Dirichlet: 反転で発散するため反転防止に使える。OptCuts/SLIM の歪み上限の単位。
- Boundary straightness (S_rect)、packing utilization (η_pack)、fragments/islands 数、overlap 率、seam length: 人手風・編集性の代理指標。DreamUV/SeamCrafter/MeshTailor の表で併用。
- 比較時は同一メッシュ・同一チャート数/歪み上限での統制が必要 (OptCuts スライドの AutoCuts 比較、MeshTailor の xatlas 比較等が例)。

---

## 参考文献・リンク集 (再掲・主要のみ)

- LSCM (2002): https://www.cs.jhu.edu/~misha/Fall09/Levy02.pdf
- ABF++ (2005): https://www.cs.ubc.ca/~sheffa/papers/abf_plus_plus.pdf
- ARAP (2007): https://igl.ethz.ch/projects/ARAP/
- BFF (2017): https://arxiv.org/abs/1704.06873
- SLIM (2017): https://igl.ethz.ch/projects/slim
- VSA (2004): https://www.geometry.caltech.edu/pubs/CAD04.pdf
- D-Charts (2005): https://www.cs.ubc.ca/labs/imager/tr/2005/Vlad_DCharts/EG05.pdf
- AutoCuts (2017): https://cims.nyu.edu/gcl/papers/Autocuts-2017.pdf
- OptCuts (2018): https://www.cs.ubc.ca/labs/imager/tr/2018/OptCuts
- xatlas: https://github.com/jpcy/xatlas
- UVAtlas: https://github.com/microsoft/UVAtlas
- Blender UV: https://docs.blender.org/manual/en/latest/modeling/meshes/editing/uv.html
- libigl tutorial: https://libigl.github.io/tutorial
- CGAL param: https://doc.cgal.org/latest/Surface_mesh_parameterization/index.html
- Ptex paper: https://ptex.us/ptexpaper.html / code https://github.com/wdas/ptex
- Nuvo: https://arxiv.org/abs/2312.05283
- FAM: https://arxiv.org/abs/2405.14633 / code https://github.com/keeganhk/FlattenAnything
- SeamGPT: https://arxiv.org/abs/2506.18017
- SeamCrafter: https://arxiv.org/abs/2509.20725
- ArtUV: https://arxiv.org/abs/2509.20710
- DreamUV: https://arxiv.org/abs/2606.22445
- SeamGen: https://arxiv.org/html/2607.12379v1
- MeshTailor: https://arxiv.org/pdf/2603.27309
- サーベイ: https://graphics.stanford.edu/courses/cs468-05-fall/Papers/param-survey.pdf / https://www.cs.ubc.ca/~sheffa/papers/param_survey06.pdf
