# 3Dリトポロジー手法 総合調査レポート

- 作成日: 2026-09-28
- 目的: 3Dリトポロジー／リメッシュ手法をなるべく多く網羅し、手法ごとに特徴・位置づけ（他との比較）・処理量（処理速度）・論文URL・実装URLをまとめる
- 調査方法: Web検索＋公式ページ／論文／GitHubの実在確認。ベンチマーク数値は論文・公式の公称値のみ採用。不明なものは「明記なし」とした

## 目次

1. [全体マップ](#1-全体マップ)
2. [古典的 簡略化・三角リメッシュ](#2-古典的-簡略化三角リメッシュ)
3. [Quadリメッシュ（方向場・パラメトリゼーション系）](#3-quadリメッシュ方向場パラメトリゼーション系)
4. [Voxel / Implicit・修復系](#4-voxel--implicit修復系)
5. [DCC・商用ツール](#5-dcc商用ツール)
6. [深層学習ベース手法](#6-深層学習ベース手法)
7. [ライブラリ総括と使い分け](#7-ライブラリ総括と使い分け)

---

## 1. 全体マップ

| 分類 | 主な手法 | 出力 | 自動化 |
|---|---|---|---|
| 簡略化 | QEM, Progressive Meshes, Decimation Master, Balancer, Blender Decimate | 三角 | 自動 |
| 等方三角リメッシュ | Botsch法, CVT/Lloyd/ODT, CGAL isotropic, MMG, Geogram | 三角 | 自動 |
| 異方性・適応 | Alliez異方性, Frame Fields, MMG anisotropic, CGAL anisotropic | 三角/四角混在 | 自動 |
| 体積→表面 | Marching Cubes, Dual Contouring, Surface Nets, Manifold DC, Dual Marching Cubes, OpenVDB VolumeToMesh | 三角/四角 | 自動 |
| 体積メッシュ | Delaunay refinement, TetWild/fTetWild, PolyCube→Hex | 四面体/Hex | 自動 |
| Quad自動リトポ | Instant Meshes, QuadriFlow, MIQ, QuadCover, QuadWild, ZRemesher, Quad Remesher, AUTOPO | 四角 | 自動〜半自動 |
| 方向場・中間層 | Knöppel方向場, ARAP, LSCM, BFF, PolyVector | 場/UV | 半自動 |
| 構造ベース | PolyCube, Skeleton (SQM), TopoTransfer | 四角/Hex | 半自動 |
| 手動 | Quad Draw, TopoGun, TopoBuild/PolyDraw, Blender手動リトポ | 四角 | 手動 |
| 微分可能抽出 | DMTet, FlexiCubes, MeshSDF, SAP, DMC, NMC, NDC/UNDC | 三角 | 最適化＋学習 |
| 学習生成 | AtlasNet, 3D-CODED, Pixel2Mesh(++), Point2Mesh, MeshGPT, MeshAnything, TripoSR, SF3D | 三角/四角風 | 学習済み推論 |
| メッシュ上学習 | MeshCNN, SubdivNet, Neural Subdivision, ROAR | 三角 | 学習/最適化 |

定石パイプライン（古典）:

```text
汚い入力 → Manifold/ManifoldPlus/MeshFix/OpenVDB で watertight化
  → MC/DC/MDC で等値面化
  → Botsch / CVT / MMG で等方化
  → Instant Meshes / QuadriFlow / ZRemesher / Quad Remesher で四角化
  → RizomUV で UV → ベイク
```

定石パイプライン（深層学習・画像→アセット）:

```text
単画像 → TripoSR / SF3D (0.5秒) → MeshAnything V2 で職人風メッシュ化 (30〜45秒)
多視点 → Instant NGP / nvdiffrec(DMTet/FlexiCubes) で最適化 → 低poly抽出
点群 → SAP (高速) / NDC・UNDC (学習済み再構成) / Point2Mesh (単形状overfit)
```

---

## 2. 古典的 簡略化・三角リメッシュ

### 2.1 Quadric Error Metrics (QEM, Garland & Heckbert 1997)

- **特徴:** 各頂点に平面までの二乗距離を表す4x4対称行列Qを持たせ、辺収縮コストを `v^T Q v` で評価。ヒープで最小コストから収縮。非多様体・非連結成分の結合も可能。属性拡張（色・テクスチャ）は1998年版。
- **位置づけ:** 簡略化のデファクト。Progressive Meshes（連続LOD）に対し「単一解像度を速く作る」用途。実装が容易で全DCC・ライブラリに浸透。Dual ContouringのQEF頂点配置にも同型アイデア。
- **処理量:** `O(n log n)`（ヒープ）、メモリ `O(n)`。当時ワークステーションで数万〜数十万面を数秒〜数十秒。現代は100k面→10k面がms〜数百msオーダー（libigl/PMP）。
- **論文:**
  - https://dl.acm.org/doi/10.1145/258734.258849
  - https://cseweb.ucsd.edu/~ravir/190/2016/garland97.pdf
  - https://www.cs.princeton.edu/courses/archive/fall03/cs526/papers/garland97.pdf
- **実装:**
  - https://github.com/libigl/libigl (`igl::decimate`)
  - https://github.com/pmp-library/pmp-library
  - https://github.com/CGAL/cgal (Surface_mesh_simplification)
  - 解説: https://www.pmp-library.org/remeshing.html

### 2.2 Progressive Meshes (PM, Hoppe 1996)

- **特徴:** `edge collapse → vsplit`列としてメッシュを可逆圧縮。エネルギー最小化で簡略列を構築し、geomorph・選択的詳細化・圧縮・プログレッシブ伝送を統一。
- **位置づけ:** QEMが「単品簡略化」に対しPMは「連続解像度表現」。View-dependent LOD・ストリーミングの元祖。構築は重いが再生は `O(1)/vsplit`。
- **処理量:** 構築は非線形最適化のためQEMより1〜2桁遅い（当時、数千面で数分）。再生・geomorphは線形。
- **論文:**
  - https://dl.acm.org/doi/10.1145/237170.237216
  - https://hhoppe.com/proj/pm/
- **実装:** 公式OSSなし。DirectX D3DX・three.js `SimplifyModifier`等に派生。現代はQEM＋vsplit記録で代替。

### 2.3 Isotropic remeshing — Botsch法 (Botsch & Kobbelt 2004, PMP)

- **特徴:** 目標辺長Lに対し反復：(1) `>4/3L`分割・`<4/5L`縮約、(2) 価数改善フリップ（内6・境界4）、(3) 接平面relaxation＋原曲面へ再投影。Dunyachら2013で曲率適応サイズ場 `L(x)` に拡張。
- **位置づけ:** 最も簡単・堅牢な等方リメッシュのデファクト。CVT/ODTの大域最適化に対し局所操作のみで高速。異方性は出せない。
- **処理量:** 1反復 `O(n)`、通常5〜10反復。10k〜100k面で対話速度。SF3Dのリメッシャオプションにも採用。
- **論文・解説:**
  - https://www.pmp-library.org/remeshing.html
  - https://cg.cs.tu-dortmund.de/publications/2013-remeshing.pdf
  - https://stanford-cs248.github.io/Cardinal3D/meshedit/global/remesh/
  - 先行: https://dl.acm.org/doi/10.1145/566570.566588 / https://www.geometry.caltech.edu/pubs/AMD02.pdf (Alliez-Meyer-Desbrun 2002)
- **実装:**
  - https://github.com/pmp-library/pmp-library (`uniform_remeshing`, `adaptive_remeshing`)
  - https://gpytoolbox.org/latest/remesh_botsch/

### 2.4 CVT / Lloyd relaxation / ODT

- **特徴:**
  - CVT: Voronoiセル重心＝サイト。正六角形漸近、良質等方三角の双対。
  - Lloyd: Voronoi構築→重心移動の固定点反復。単純だが線形収束。
  - ODT (Chen & Xu 2004): 線形補間誤差最小のDelaunay。3Dでsliver除去に強い。
  - 高速化：L-BFGS/P-L-BFGS、GeogramのNewton法。Restricted Voronoi (Yanら2009) でパラメータ化不要。
- **位置づけ:** Botsch局所法より大域最適で三角形品質・ブルーノイズ性が高い。密度制御・異方性（6D CVT）まで拡張可。
- **処理量:** 1 Lloyd反復はRestricted Voronoi `O(n log n)`支配。L-BFGS/Newtonで反復数を1/2〜1/10に削減。低discrepancy初期化で総時間約50%減の報告あり。
- **論文:**
  - Alliezら等方リメッシュ: https://inria.hal.science/hal-00787166 / https://ligm.univ-eiffel.fr/~colinde/pub/02remesh.pdf
  - ODT: https://www.math.uci.edu/~chenlong/CX2004.html / https://computmath.cjoe.ac.cn/jcm/EN/Y2004/V22/I2/299
- **実装:**
  - https://github.com/BrunoLevy/geogram
  - https://github.com/BrunoLevy/geogram/wiki/Remeshing
  - https://doc.cgal.org/latest/Mesh_3/index.html

### 2.5 Anisotropic remeshing (Alliezら2003, Bossen-Heckbert)

- **特徴:** 曲率テンソルを計量場として曲率線に沿った伸長要素を生成。円筒・フィレット・CAD形状を少数要素で近似。AlliezらSIGGRAPH03は曲率線追跡＋等方域点サンプリングでquad-dominant化。
- **位置づけ:** 等方では要素数爆発する形状向け。FEM境界層・Lp最適近似の双方で要請。実装・ロバスト性は等方より一段難しい。
- **処理量:** 曲率推定＋方向場平滑＋線積分が支配的。厳密計算量の公表なし。
- **論文:**
  - https://dl.acm.org/doi/10.1145/1201775.882296
  - https://www.geometry.caltech.edu/pubs/ACDLD03.pdf
  - https://inria.hal.science/inria-00071778/file/RR-4808.pdf
  - 拡張: http://mesh.brown.edu/dgp/pdfs/Marinov-pg2004.pdf
- **実装:**
  - https://github.com/BrunoLevy/geogram/wiki/Remeshing (6D CVT異方性)
  - https://github.com/MmgTools/mmg
  - https://doc.cgal.org/latest/Mesh_3/index.html

### 2.6 Delaunay refinement / TetWild / fTetWild

- **特徴:**
  - Delaunay refinement (Ruppert/Chew/Shewchuk): 悪要素の外心挿入。角度・grading・サイズ最適性の理論保証。Triangle/TetGen/CGAL Mesh_3の核。
  - TetWild (SIGGRAPH18): triangle soup→四面体。exact rational＋ε-envelopeで自己交差・穴を吸収。
  - fTetWild (SIGGRAPH20): floatのみ増分挿入＋並列化。TetWild同等品質で10倍速級。
- **位置づけ:** 古典Delaunayは「きれいな入力→保証付き」、Wild系は「汚い入力→とにかくvalidな体積」。表面リメッシュと体積メッシュの中間。
- **処理量:** Delaunay refinementはサイズ最適（定数倍）。Wild系はThingi10k 10kモデルで評価。TetWildは単スレッドで遅く100GB要例あり、fTetWildはDelaunay法並みに高速化。デフォルト `l=b/20, ε=b/1000`。
- **論文:**
  - https://dl.acm.org/doi/10.1145/3197517.3201353 (TetWild)
  - https://dl.acm.org/doi/10.1145/3386569.3392385 (fTetWild)
  - https://web.uvic.ca/~teseo/profile/publications/ftetwild/2020-fTetWild.pdf
  - https://people.eecs.berkeley.edu/~jrs/papers/triangle.pdf
  - https://people.eecs.berkeley.edu/~jrs/papers/2dj.pdf
- **実装:**
  - https://github.com/Yixin-Hu/TetWild
  - https://github.com/wildmeshing/fTetWild
  - https://wildmeshing.github.io/ftetwild/
  - https://github.com/CGAL/cgal

### 2.7 Marching Cubes / Dual Contouring / Surface Nets / Manifold DC

- **特徴:**
  - Marching Cubes (1987): 256ケーステーブル。manifold・watertightだが特徴が鈍り悪い三角形多。
  - Surface Nets (Gibson 1998): セル内1頂点＋符号変化辺でquad結合。MCの双対で均整良。
  - Dual Contouring (Juら2002): Hermiteデータ＋QEF最小化でシャープ再現。octree適応・crack-free。ただし非多様体が出る。
  - Manifold DC (Schaeferら2007) / Dual Marching Cubes (Nielson 2004): 多様体保証＋特徴保持。
- **位置づけ:** 体積→表面の定番。MC＝単純・堅牢、DC系＝適応・特徴保持、MDC＝後処理に耐える多様体が必要な場合。
- **処理量:** いずれもボクセル数 `O(N)`＋テーブル参照。DCはセル毎3x3 QEFの定数増のみ。適応octreeで出力面数大幅削減。
- **論文:**
  - https://dl.acm.org/doi/10.1145/37401.37422 (MC)
  - https://people.eecs.berkeley.edu/~jrs/meshpapers/LorensenCline.pdf
  - https://www.cs.rice.edu/~jwarren/papers/dualcontour.pdf (DC)
  - https://dl.acm.org/doi/10.1145/566570.566586
  - https://people.engr.tamu.edu/schaefer/research/dualsimp_tvcg.pdf (MDC)
  - https://merl.com/publications/docs/TR99-24.pdf (Surface Nets)
  - https://pmc.ncbi.nlm.nih.gov/articles/PMC9623606/ (Surface Nets解説)
- **実装:** VTK `vtkMarchingCubes`、OpenVDB、scikit-image `marching_cubes`。DC/MDCはGeogram・MMGの等値面機能で代替が実用的。

---

## 3. Quadリメッシュ（方向場・パラメトリゼーション系）

### 3.1 Instant Meshes (Jakobら2015)

- **特徴:** orientation field＋position fieldを局所平滑化オペレータで交互最適化。入力は三角メッシュ・点群・スキャン可。境界・シャープエッジスナップ、対称性オプション。出力はquad-dominant（三角残存あり）。
- **位置づけ:** 速度最速クラス。QuadriFlowのベース。特異点が多いのが弱点でSubdiv・パラメトリゼーション用途では不利。
- **処理量:** ほぼ線形。数十万faceで1秒未満。372M三角のSt. Matthew像で9分18秒。SGP Outstanding Software Award 2020、SIGGRAPH Asia Test of Time Award 2025。
- **論文:**
  - https://dl.acm.org/doi/10.1145/2816795.2818078
  - https://igl.ethz.ch/projects/instant-meshes
  - https://rgl.epfl.ch/publications/Jakob2015Instant
- **実装:**
  - https://github.com/wjakob/instant-meshes
  - https://pypi.org/project/pyinstantmeshes

### 3.2 QuadriFlow (Huangら2018)

- **特徴:** Instant Meshesのposition field特異点を最小コストネットワークフロー＋局所SATでグローバル除去。orientation fieldは温存。watertight純粋quad指向。
- **位置づけ:** 速度はInstant Meshesより遅いがMIQより大幅に高速。特異点はInstant Meshesの約1/4。閉多様体主対象で境界・非多様体に弱い。SGP 2018 Best Paper。
- **処理量:** ShapeNet由来17,791モデルで1モデル10秒未満。
- **論文:**
  - https://onlinelibrary.wiley.com/doi/10.1111/cgf.13498
  - https://web.stanford.edu/~jingweih/papers/quadriflow
  - https://www.niessnerlab.org/papers/2018/10quadriflow/huang2018quadriflow.pdf
- **実装:**
  - https://github.com/hjwdzh/QuadriFlow (MIT)
  - https://github.com/hjwdzh/Manifold (前処理)
  - Blender統合: https://github.com/blender/blender/blob/main/extern/quadriflow/README.md

### 3.3 MIQ — Mixed-Integer Quadrangulation (Bommesら2009)

- **特徴:** field-guided quad meshingの祖型。(1) cross field生成、(2) グローバルパラメトリゼーションを混合整数問題としてgreedy近似。ユーザ制約（方向・特異点）を取り込める。
- **位置づけ:** 品質の参照点。以降の品質上限。速度は大幅に低速（分オーダー）。開放境界・CAD堅牢性はQuadWild等が上。
- **処理量:** NP困難クラスをgreedy近似。中規模で分オーダー（原論文Table 1参照）。
- **論文:**
  - https://dl.acm.org/doi/10.1145/1531326.1531383
  - https://www.graphics.rwth-aachen.de/media/papers/bommes_zimmer_2009_siggraph_011.pdf
  - https://www.graphics.rwth-aachen.de/publication/0344
- **実装:**
  - https://github.com/libigl/libigl (tutorial §505 integer-grid parametrization)
  - https://github.com/hcebke/libQEx (quad抽出)

### 3.4 Globally Optimal Direction Fields / QuadCover / Frame Fields

#### (a) Knöppelら2013 方向場

- **特徴:** n方向場を二次平滑エネルギーで定式化し、全特異点配置に対する大域最適解を疎固有値問題（ガイダンスありはPoisson1回）で求める。従来比1桁高速。
- **位置づけ:** 方向場生成の標準理論。MIQのcross field生成を置き換え可。
- **論文:**
  - https://dl.acm.org/doi/10.1145/2461912.2462005
  - https://www.cs.cmu.edu/~kmcrane/Projects/GloballyOptimalDirectionFields
  - https://www.cs.cmu.edu/~kmcrane/Projects/GloballyOptimalDirectionFields/paper.pdf
- **実装:**
  - https://github.com/avaxman/Directional

#### (b) QuadCover (Kälbererら2007)

- **特徴:** frame fieldを分岐被覆上の単一ベクトル場に変換しHodge分解で積分可能化。MIQの直接先行手法。3D拡張がCubeCover。
- **位置づけ:** MIQ登場後は参照点化。libQExがQuadCover出力を想定する等エコシステム重要性は継続。
- **論文:** http://polthier.info/articles/quadCover/KNP07-QuadCover.pdf (CGF 26(3))
- **実装:** libQEx・libigl/Directional内再実装を利用（単体公式リポジトリなし）。

#### (c) Frame Fields (Panozzoら2014)

- **特徴:** cross fieldを非直交・非単位長に一般化し異方性・非一様密度を表現。変形→等方quadrangulation→逆変形で既存手法を異方性対応化。
- **位置づけ:** 等方手法に密度・異方性制御を付与する拡張層。CAD・シミュレーション向け。
- **論文:**
  - https://dl.acm.org/doi/10.1145/2601097.2601179
  - https://igl.ethz.ch/projects/frame-fields/
- **実装:** https://github.com/libigl/libigl (tutorial §506)

### 3.5 パラメトリゼーション中間層: ARAP / LSCM / BFF

| 手法 | 特徴 | 位置づけ | 処理量 | 論文 | 実装 |
|---|---|---|---|---|---|
| ARAP (Sorkine & Alexa 2007) | 局所剛性保持のlocal-global反復。変形・パラメトリゼーション両用 | 変形・UVの中間層 | 疎線形solve反復。数十万要素で対話的 | https://igl.ethz.ch/projects/ARAP/ | https://github.com/libigl/libigl/blob/main/include/igl/arap.h |
| LSCM (Lévyら2002) | Cauchy-Riemann最小二乗近似の準等角展開。2点固定で線形一発 | 高速だがfold保証なし | 線形疎最小二乗 | https://dl.acm.org/doi/10.1145/566654.566590 | libigl `igl/lscm.h` |
| BFF (Sawhney & Crane 2017) | 境界をCherrier公式＋Poincaré-Steklovで先決め。境界・錐特異点直接制御 | 境界制御◎ | Laplace factorization1回。既存比償却1/50。数百万triで対話可 | https://arxiv.org/abs/1704.06873 / https://dl.acm.org/doi/10.1145/3132705 | https://github.com/GeometryCollective/boundary-first-flattening |

### 3.6 PolyCube系 / Skeletonベース

#### PolyCube系

- **特徴:** 軸平行箱の合併との低歪み写像で正則hexグリッド転写。表面抽出でquadにも。PolyCube-Maps (2004) → PolyCut (2013) → CE-PolyCube (2020) → Robust Quantization (2022) → Evocube (2022)。
- **位置づけ:** CAD・FEMのblock構造・内部特異点制御に強い。汎用有機形状の全自動化は難しくlabelingがボトルネック。表面quad用途ではfield系より手数。
- **処理量:** labeling＋変形が支配的。モデル依存大。
- **論文・実装:**
  - https://dl.acm.org/doi/10.1145/1015706.1015810
  - https://www.cs.ubc.ca/~sheffa/hexing/hexing.htm
  - https://github.com/msraig/CE-PolyCube / https://guohaoxiang.github.io/projects/ce_polycube.html
  - https://github.com/fprotais/robustPolycube
  - https://github.com/LIHPC-Computational-Geometry/evocube

#### Skeletonベース (SQM等)

- **特徴:** curve skeletonから管状接続性を起こしquad/hex生成。管状・分岐形状（血管・四肢）で破綻しにくい。前処理にMean Curvature Skeleton・CGAL骨格抽出。
- **位置づけ:** 管状・有機形状の堅牢性は高いが一般形状の面品質・特異点制御はfield系に劣る。
- **論文・実装:**
  - https://www.sciencedirect.com/science/article/pii/S0097849312000556
  - https://onlinelibrary.wiley.com/doi/abs/10.1111/j.1467-8659.2012.03178.x
  - https://doc.cgal.org/latest/Surface_mesh_skeletonization/index.html
  - https://onlinelibrary.wiley.com/doi/10.1111/cgf.13021

### 3.7 QuadWild (PietroniらSIGGRAPH 2021) — 番外の注目株

- **特徴:** フィーチャ線駆動の半正則quadリメッシュ。Thingi10k全体バッチで失敗率0.5%未満の堅牢性。CAD向け現状最良選択肢の一つ。
- **位置づけ:** MIQ品質＋QuadriFlow速度＋CAD堅牢性を狙う。フィーチャ保持が要るCAD・ハードサーフェス向け。
- **処理量:** 大規模バッチ検証済み。個別msは論文参照。
- **論文:** https://doi.org/10.1145/3450626.3459941
- **実装:** https://github.com/nicopietroni/quadwild (GPL)

### 3.8 Quadリメッシュ比較表

| 手法 | 速度 | 品質（特異点） | 対称性 | 境界対応 | 主用途 |
|---|---|---|---|---|---|
| Instant Meshes | ◎（数十万face<1秒） | ○（多め） | ○ | ○ | 対話的汎用 |
| QuadriFlow | ○（<10秒） | ◎（IMの1/4） | △ | △ | Subdiv最終トポ |
| MIQ | △（分） | ◎（参照） | △ | ○ | 高品質研究基準 |
| Knöppel場 | ◎（従来比1桁高速） | ◎ | — | ○ | 方向場土台 |
| QuadCover | ○ | ○ | △ | ○ | パラメ経由 |
| Frame Fields | ○ | ◎（異方性） | △ | ○ | 適応メッシュ |
| PolyCube | △〜○ | ◎（block） | △ | △ | CAD・FEM hex |
| QuadWild | ○ | ◎ | △ | ○ | CAD堅牢quad |

---

## 4. Voxel / Implicit・修復系

### 4.1 OpenVDB

- **特徴:** DreamWorks発・ASWF管理の疎ボクセルライブラリ。`VolumeToMesh`でSDF→メッシュ化。Blender Voxel Remesh・3ds Max OpenVDB Remeshの基盤。TBB並列。
- **位置づけ:** 自動・三角主体（適応で四角混在可）・Apache-2.0・C++ライブラリ。Quad専用リメッシャーはなし。
- **処理量:** 解像度依存。数百万ボクセルで数秒〜数十秒。
- **URL:**
  - https://www.openvdb.org/
  - https://www.openvdb.org/documentation
  - https://www.openvdb.org/documentation/doxygen
  - https://github.com/AcademySoftwareFoundation/openvdb

### 4.2 Dual Marching Cubes (Schaefer & Warren 2004 / Nielson 2004)

- **特徴:** octree双対グリッド上でMCを行い薄壁を過剰分割なしに再現。クラックフリー・適応的・多様体保証。MC・DCの弱点補完。
- **位置づけ:** 自動・三角/四角（実装依存）・OSS実装あり。
- **処理量:** octree深さ依存。小〜中規模でリアルタイム〜数秒。
- **論文・実装:**
  - https://people.eecs.berkeley.edu/~jrs/meshpapers/SchaeferWarren.pdf
  - https://github.com/dominikwodniok/dualmc

### 4.3 Manifold / ManifoldPlus (Huangら)

- **特徴:** triangle soup→watertight manifoldへの堅牢変換。`--depth 8`等で解像度制御。Eigen＋libigl依存。非商用のみ無料ライセンス注意。
- **位置づけ:** 自動・三角多様体出力。ShapeNet 17kモデル級で検証。
- **処理量:** 数万faceで秒オーダー。
- **URL:**
  - https://github.com/hjwdzh/ManifoldPlus
  - https://github.com/hjwdzh/Manifold
  - https://arxiv.org/abs/2005.11621

### 4.4 MeshFix (Attene, CNR-IMATI)

- **特徴:** スキャンメッシュの穴・自己交差・縮退・非多様体を除去し単一watertight化。正常部非改変が特徴。PyMeshFixラッパーあり。
- **位置づけ:** 自動（半自動パラメータあり）・三角・GPL。
- **処理量:** 数十万〜百万polyで数秒〜1分。
- **URL:**
  - https://github.com/MarcoAttene/MeshFix-V2.1
  - https://github.com/pyvista/pymeshfix
  - https://pymeshfix.pyvista.org/
  - http://www.meshrepair.org/

---

## 5. DCC・商用ツール

### 5.1 Blender — Voxel Remesh / QuadriFlow Remesh / Decimate

- **Voxel Remesh:** OpenVDBでwatertight等方均一化。自己交差・内部解消に強い。`Voxel Size/Adaptivity/Fix Poles/Preserve Volume`。出力は三角形。スカルプト中間用でアニメ用エッジフローにはならない。「QuadriFlowは低速だが最終トポ向け高品質」と公式位置づけ。数十万〜数百万polyで数秒〜十数秒。
  - https://docs.blender.org/manual/en/latest/sculpt_paint/sculpting/tool_settings/remesh.html
  - https://docs.blender.org/manual/en/latest/modeling/meshes/retopology.html
  - https://docs.blender.org/manual/en/latest/modeling/modifiers/generate/remesh.html
- **Quadriflow Remesh:** §3.2を `extern/quadriflow` として統合。`Target Faces`指定。非多様体・Dyntopo出力を拒否しがちでVoxelで修復→Quadriflowが定石。数万quadで数十秒〜数分・フリーズ報告あり。
  - https://github.com/blender/blender/blob/main/extern/quadriflow/README.md
  - https://www.blender.org/download/releases/2-81
  - https://devtalk.blender.org/t/quadriflow-integration/23156
- **Decimate Modifier:** `Collapse/Un-Subdivide/Planar`の3mode。軽量化用。即時〜数秒。
  - https://docs.blender.org/manual/en/latest/modeling/modifiers/generate/decimate.html

### 5.2 ZBrush — ZRemesher / Dynamesh / Decimation Master

- **ZRemesher:** ワンクリック四角自動リトポの事実上標準。`Target/Adapt/Keep Creases/Detect Edges/Freeze Border/Guides/Polypaint Density/Symmetry`。内部アルゴリズム非公開。v4.0で高速化。200k→10kを数秒と公式例。
  - https://help.maxon.net/zbr/en-us/Content/html/user-guide/3d-modeling/topology/zremesher/zremesher-4-0/zremesher-4-0.html
  - https://help.maxon.net/zbr/en-us/Content/html/reference-guide/tool/polymesh/geometry/zremesher/zremesher.html
  - https://www.maxon.net/en/zbrush/features/remeshing-options
- **Dynamesh:** 粘土的造形用ボクセル的再分配。Ctrl+ドラッグ更新。数百万polyで1秒未満〜数秒と公式は「fraction of second」と記述。最終トポではない。
  - https://help.maxon.net/zbr/en-us/Content/html/user-guide/3d-modeling/modeling-basics/creating-meshes/dynamesh/dynamesh.html
- **Decimation Master:** ディテール保持削減プラグイン。UV/Polypaint/Mask/ボーダー保護。ZRemesher（再構成）とは別物。2.2M pointsで30〜50秒報告。20M→2M級も可。
  - https://help.maxon.net/zbr/en-us/Content/html/user-guide/zbrush-plugins/decimation-master/decimation-master.html

### 5.3 Maya Retopo (Quad Draw + Retopologize)

- **特徴:** 手動 `Quad Draw`（Live拘束・Relaxブラシ）＋自動 `Mesh > Retopologize`（2023前処理統合、2024 Symmetry・角度/タグ別特徴保持・非多様体チェック追加）。
- **位置づけ:** 手動＋自動・四角・商用。
- **処理量:** 自動は中規模で数十秒〜数分。手動は作業者依存。
- **URL:**
  - https://help.autodesk.com/view/MAYAUL/2027/ENU?guid=GUID-74867E07-5CCD-4D55-B60B-C90C3AD65DF4
  - https://help.autodesk.com/cloudhelp/2025/ENU/Maya-Modeling/files/GUID-74867E07-5CCD-4D55-B60B-C90C3AD65DF4.htm

### 5.4 3ds Max Retopology (ReForm + OpenVDB Remesh)

- **特徴:** ReForm自動四角化＋前処理/OpenVDB分離（v1.5）。Slice/Smooth/ProOptimize/Mesh Cleaner前処理、Weighted Normals/Symmetry後処理が推奨。2026で改善。
- **位置づけ:** 自動・四角（＋Voxel三角前処理）・商用同梱。
- **処理量:** v1.5で最大8%高速化と公表。中規模で数十秒〜数分。
- **URL:**
  - https://help.autodesk.com/cloudhelp/2026/ENU/3DSMax-Retopology/files/GUID-57B77A00-5300-47CB-99E1-22B9C536B060.html

### 5.5 TopoGun 3 / 3D-Coat AUTOPO / Houdini / MeshLab / Meshmixer / Balancer / Quad Remesher / RizomUV

- **TopoGun 3 (Pixelmachine):** 手動リトポ＋Patch半自動面埋め特化老舗。一括ベイク（256コア・AOはGPU）。商用perpetual。
  - https://www.topogun.com/ / http://docs.topogun.com/
- **3D-Coat AUTOPO:** スカルプト→Retopo Room自動四角化。対称必須推奨・ガイドストローク・密度ペイント。平均モデル30秒〜数分。
  - https://3dcoat.com/features/retopo-tools / https://3dcoat.com/documentation/manual/workspaces-rooms/retopology/autoretopo
- **Houdini:** `Quad Remesh`（H21適応・シャープ保持強化）＋手動 `TopoBuild/PolyDraw`＋ `TopoTransfer/TopoFlow`。完全非破壊・再実行可。中規模で数秒〜数分。LabsはOSS。
  - https://www.sidefx.com/products/houdini/modeling/retopology
  - https://github.com/sideeffects/SideFXLabs
- **MeshLab (ISTI-CNR):** 三角処理OSS定番。Quadric Edge Collapse・Cleaning・Poisson等数百フィルタ。PyMeshLabで自動化。数百万face簡略化で数秒〜数十秒。
  - http://www.meshlab.org/ / https://github.com/cnr-isti-vclab/meshlab
- **Meshmixer (Autodesk, 開発停止):** `Make Solid` voxel的watertight化・Remesh/Reduce・Inspector修復。数十万polyで即時〜数秒。Fusionへ移行推奨。
  - https://www.research.autodesk.com/projects/meshmixer
- **Atangeo Balancer:** 高忠実削減特化。法線・UV・マテリアル・境界保持、LOD一括、watertight検証。数百万faceを迅速・低メモリ。
  - https://atangeo.com/ / https://atangeo.com/products / https://atangeo.com/guide
- **Exoside Quad Remesher:** ワンクリック曲率追従四角化。`Target Quads/Adaptive/対称/ガイド/ハードエッジ保持`。Blender/Maya/Max/Houdini/Modo対応。非公開プロプライエタリ。ZRemesher系譜の商用後継として標準化。中規模数万quadで数秒〜1分。
  - http://exoside.com/ / https://exoside.com/quadremesher/quadremesher-doc/
  - https://www.exoside.com/quadremesherdata/QuadRemesher_1.2_UserDoc.pdf
- **RizomUV:** リトポ自体は行わないが後のUV展開・パッキング特化。リトポ品質がUV歪みに直結するため対で評価。50万アイランド1分未満（GPU Packing公称）。
  - https://www.rizomuv.com/

**DCC使い分け:** スキャン修復→Voxel/Dynamesh/Manifold/MeshFix。アニメ用四角→ZRemesher/Quad Remesher/QuadriFlow/Maya/Max/Houdini/TOPOGUN/AUTOPO。削減のみ→Decimation Master/Decimate/Balancer/MeshLab。

---

## 6. 深層学習ベース手法

### 6.1 微分可能アイソサーフェス抽出系

#### DMTet — Deep Marching Tetrahedra (ShenらNeurIPS 2021)

- **特徴:** 変形可能四面体上の離散SDF＋微分可能MT層＋細分割。表面損失を直接最適化。任意トポロジー可。GET3D・nvdiffrecの標準バックボーン。
- **位置づけ:** 微分可能○。学習要（coarse voxel/点群→高解像度）。入力はvoxel・点群・画像特徴。
- **処理量:** coarse-to-fineでメモリ削減。絶対ms明記なし。`64^3 FlexiCubes ≒ 80^3 DMTet`の三角数。
- **論文:** https://arxiv.org/abs/2111.04276
- **実装:**
  - https://github.com/nv-tlabs/DMTet
  - https://github.com/NVIDIAGameWorks/kaolin
  - https://github.com/NVlabs/nvdiffrec
  - https://github.com/nv-tlabs/GET3D

#### FlexiCubes (ShenらSIGGRAPH 2023)

- **特徴:** DMTet後継。Dual Marching Cubesベースにcube毎21パラメータ等を追加し勾配最適化で接続調整。nvdiffrecのdrop-in replacement。tetra・階層adaptive版あり。
- **位置づけ:** 微分可能○。最適化モジュール単体でも可。入力はvoxel SDF＋per-cubeパラメータ。
- **処理量:** 同解像度ではDMTetよりdenseで重い。推奨 voxel:tetra=4:5。絶対ms明記なし。
- **論文:** https://arxiv.org/abs/2308.05371
- **実装:** https://github.com/nv-tlabs/FlexiCubes

#### Neural Marching Cubes (Chen & Zhang, SIGGRAPH Asia 2021)

- **特徴:** 初の学習ベースMC。trilinear仮定を捨て近傍文脈を3D CNNで学習。1 cube 23頂点パラメータ化。sharp復元。
- **位置づけ:** 推論時は学習済みフィードフォワード。学習要（ABC/CAD）。入力はSDF/occupancy voxel。
- **処理量:** 従来MC比4-8x三角数・100x以上compute（NDC論文比較）。
- **論文:** https://arxiv.org/abs/2106.11272
- **実装:** https://github.com/czq142857/NMC

#### NDC / UNDC — Neural Dual Contouring (ChenらSIGGRAPH 2022)

- **特徴:** DCを学習化。1 cell 1頂点＋交差辺毎1 quadでsharpを効率表現。NDCは符号予測（watertight）、UNDCは符号なし（開曲面・薄肉・布可）。
- **位置づけ:** 学習要（3D CNN）。入力はSDF/UDF/voxel/無向点群・ノイズスキャンと統一的。
- **処理量:** NMC比で三角数1/4〜1/8、推論1/3〜1/20。ConvONet・Poisson・SIREN・LIGより精度・速度・品質で優位と報告。
- **論文:** https://arxiv.org/abs/2202.01999
- **実装:** https://github.com/czq142857/NDC

#### SAP — Shape As Points (PengらNeurIPS 2021 Oral)

- **特徴:** 有向点群↔meshを微分可能Poisson Solver（FFT・GPU）で接続。watertight・manifold・任意トポロジー。軽量・初期化容易。
- **位置づけ:** 微分可能○。最適化のみでも可＋学習再構成可。ノイズ・外れ値に強い。
- **処理量:** 推論 R2N2 15ms・AtlasNet 25ms・ConvONet 327ms・SAP 64ms。最適化 IGR ~30min・SAP ~6min・SPSR ~9sec。cubic memoryで大規模苦手。
- **論文:** https://arxiv.org/abs/2106.03452
- **実装:** https://github.com/autonomousvision/shape_as_points

#### MeshSDF / DeepMesh (RemelliらNeurIPS 2020 / Guillardら2022)

- **特徴:** 連続DeepSDFからMC抽出表面サンプルの `dV/df` を閉形式微分。forward非微分MC・backwardカスタム勾配。単視点再構成＋空力最適化でSOTA超え。car 9-20%・chair 8-36%改善。
- **位置づけ:** 微分可能○。DeepSDF事前学習＋潜在精錬要。
- **論文:** https://arxiv.org/abs/2006.03997 / https://arxiv.org/abs/2106.11795
- **実装:** https://github.com/cvlab-epfl/MeshSDF

#### Deep Marching Cubes (LiaoらCVPR 2018)

- **特徴:** end-to-end表面予測の元祖。occupancy＋頂点変位でMCを微分可能層に。TSDF中間loss不要。
- **位置づけ:** 微分可能○（単連結のみ）。低解像度前提で組合せ爆発。
- **論文:** https://openaccess.thecvf.com/content_cvpr_2018/html/Liao_Deep_Marching_Cubes_CVPR_2018_paper.html
- **実装:** 公式GitHubなし。

### 6.2 変形・パッチベース生成

- **AtlasNet (GroueixらCVPR 2018):** 2D square→3Dの集合で被覆。任意解像度サンプリング。SAP比較で25ms。https://arxiv.org/abs/1802.05384 / https://github.com/ThibaultGROUEIX/AtlasNet
- **3D-CODED (GroueixらECCV 2018):** テンプレート変形で形状＋対応同時学習。FAUST-inter 2.88cm。https://arxiv.org/abs/1806.05228 / https://github.com/ThibaultGROUEIX/3D-CODED
- **Pixel2Mesh (WangらECCV 2018):** 楕円体をG-ResNetでcoarse-to-fine変形。単RGB→2466頂点。https://arxiv.org/abs/1804.01654 / https://github.com/nywang16/Pixel2Mesh
- **Pixel2Mesh++ (WenらICCV 2019):** 多視点版。MDN加重。Titan Xp 96h学習。https://arxiv.org/abs/1908.01491 / https://github.com/walsvid/Pixel2MeshPlusPlus (コミュニティ再実装)
- **Point2Mesh HanockaらSIGGRAPH 2020 (Self-Prior):** 単一点群にCNN重みoverfitし初期meshをshrink-wrap。学習不要・test-time最適化。https://doi.org/10.1145/3386569.3392415 / https://github.com/ranahanocka/point2mesh
- **Point2Mesh LiuらECCV 2020 (IER):** KNN候補三角を分類。学習要。https://arxiv.org/abs/2007.09267 / https://github.com/Colin97/Point2Mesh

### 6.3 メッシュ上CNN・細分割学習

- **MeshCNN (HanockaらSIGGRAPH 2019):** edge conv＋edge collapse pooling。生成器でないが基準。https://arxiv.org/abs/1809.05910 / https://github.com/ranahanocka/MeshCNN
- **SubdivNet (HuらTOG 2022):** Loop細分割接続にremeshしface conv。閉多様体要remesh。https://arxiv.org/abs/2106.02285 / https://github.com/lzhengning/SubdivNet
- **Neural Subdivision (LiuらSIGGRAPH 2020):** Loop更新固定・頂点位置のみNN予測。単一高解像度学習でも汎化。https://arxiv.org/abs/2005.01819 / https://github.com/HTDerekLiu/neuralSubdiv
- **ROAR (BardaらTOG 2023):** 2D render＋Planar Projection lossで進化→適応分割反復。MC比大幅に少ないSDFサンプルで同等Chamfer。GPU prototypeでCPU比1桁高速。https://arxiv.org/abs/2307.00690 (公式GitHub確認できず)

### 6.4 NeRF・拡散からのメッシュ抽出・生成

- **Instant NGP (MüllerらSIGGRAPH 2022):** hash encoding＋fused MLP。NeRF/SDF→MC内蔵。数秒学習・1080p数十ms。NeRF/NSVFと15秒で競合。https://arxiv.org/abs/2201.05989 / https://github.com/NVlabs/instant-ngp
- **TripoSR (Tochilkinら2024):** LRM feed-forward。DINO＋triplane。A100で0.5秒以内。https://arxiv.org/abs/2403.02151 / https://github.com/VAST-AI-Research/TripoSR
- **Stable Fast 3D (Bossら2024):** TripoSR改良。DINOv2＋material予測＋高速UV。H100で0.5秒。三角数 27.4K vs TripoSR 32.1K vs InstantMesh 57.3K vs OpenLRM 662K。Botsch法オプション付き。https://arxiv.org/abs/2408.00653 / https://github.com/Stability-AI/stable-fast-3d
- **MeshAnything / V2 (ChenらICLR 2025/ICCV 2025):** 抽出を生成問題化。VQ-VAE＋transformerでArtist-Created Mesh化。数百倍面数削減。V1 A6000 7GB・30秒・800面上限、V2 8GB・45秒・1600面上限。https://arxiv.org/abs/2406.10163 / https://github.com/buaacyw/MeshAnything / https://arxiv.org/abs/2408.02555 / https://github.com/buaacyw/MeshAnythingV2
- **MeshGPT (Siddiquiら2024):** 量子化語彙→transformerで三角形列生成。coverage +9%・FID 30点改善。自己回帰で遅い。https://arxiv.org/abs/2311.15475 / https://github.com/audi/MeshGPT

**深層学習使い分け:** 高品質微分抽出→FlexiCubes＞DMTet＞MeshSDF。学習済み高速再構成→NDC/UNDC。点群watertight→SAP。粗→密→Neural Subdivision。画像→即時→TripoSR/SF3D→MeshAnythingでAM化。コンパクト生成→MeshGPT/MeshAnything（遅い）。

---

## 7. ライブラリ総括と使い分け

| ライブラリ | 得意 | URL |
|---|---|---|
| libigl | 研究プロト・QEM・パラメ・Boolean | https://github.com/libigl/libigl / https://libigl.github.io/ |
| CGAL | 保証付きDelaunay・等方・Mesh_3・Boolean | https://github.com/CGAL/cgal / https://www.cgal.org/ |
| Geogram | CVT/Newton高速・再構成・6D異方性 | https://github.com/BrunoLevy/geogram / https://brunolevy.github.io/geogram/ |
| MMG | 等方/異方・表面/体積・level-set産業級 | https://github.com/MmgTools/mmg / https://mmgtools.github.io/ |
| PMP | Botsch系最短実装・MIT | https://github.com/pmp-library/pmp-library / https://www.pmp-library.org/remeshing.html |
| TetWild/fTetWild | soup→tetブラックボックス | https://github.com/Yixin-Hu/TetWild / https://github.com/wildmeshing/fTetWild |
| OpenVDB | 疎ボリューム・Voxel Remesh基盤 | https://github.com/AcademySoftwareFoundation/openvdb |
| Kaolin/nvdiffrec | DMTet/FlexiCubes実用実装 | https://github.com/NVIDIAGameWorks/kaolin / https://github.com/NVlabs/nvdiffrec |

**選定目安:**

- とにかく速く等方化→Botsch（PMP/libigl/gpytoolbox）
- 品質・密度制御→CVT/Geogram/CGAL
- CAD・FEM異方性→MMG/Geogram anisotropic
- 汚い入力の体積化→fTetWild
- 対話的quad→Instant Meshes、最終トポ→QuadriFlow/ZRemesher/Quad Remesher/QuadWild
- 微分最適化→FlexiCubes、高速再構成→NDC/UNDC/SAP、画像→3D→TripoSR/SF3D→MeshAnything

---

*注記: ZRemesher・Quad Remesher等の商用エンジン内部アルゴリズムは非公開のため機能面からの整理とした。速度数値はすべて論文・公式の公称値。個別の実測比較は対象形状・解像度・ハードウェア依存が大きいため原論文のTable参照を推奨。*
