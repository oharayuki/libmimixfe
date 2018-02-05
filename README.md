# libmimixfe
mimi XFE module for Fairy I/O Tumbler

## はじめに

### 概要

T-01 用バイナリとして提供される libmimixfe.so とそのヘッダファイル、及び簡単な利用例を含むリポジトリです。 `include/` 以下にヘッダファイル、`lib/` 以下に libmimixfe.so 本体、`examples` 以下に利用例のソースコードが含まれます。利用例は、説明のための実装であり、プロダクション用の実装には好適ではない場合があります。

### 依存ライブラリ

LED リングの制御のために libtumbler.so が必要です（ https://github.com/FairyDevicesRD/tumbler )


### 利用例のビルド

T-01 実機上の Makefile が `examples/` 直下に用意されています。

``````````.sh
$ cd examples
$ make
``````````

適宜同梱の lixmimife.so 及び上記 libtumbler.so とリンクして実行してください。いくつかの利用例は `sudo` を必要とします。

## libmimixfe API

### API 概要

libmimixfe は Tumbler の 18ch マイクを直接入力とし、設定に従った信号処理済音声を出力するライブラリです。最も単純には、信号処理として無処理であり、その場合は、18ch マイクから同期録音された 18ch 音声がそのまま出力されます。典型的には、信号処理として、例えば、エコーキャンセル、ビームフォーミング（多チャンネルノイズ抑制）、VAD（発話区間抽出）を行った音声が出力されます。これはつまり、Tumbler のスピーカーから再生されている音声をキャンセルし（エコーキャンセル）、特定方向のみの音声を取得することで、目的音声を強調し（ビームフォーミング）、人間の声の区間のみが抽出された（VAD）、音声が取得できるということになります。

処理済み音声が出力されるタイミングで、ユーザー定義のコールバック関数が、処理済み音声を引数に与えられた状態で呼ばれます。コールバック関数にはユーザー定義のデータを与えることも可能です。この仕組は、Linux 環境での録音・再生によく用いられる OSS である portaudio （Portable Cross-Platform Audio I/O） が提供している API と同様の仕組みであり、コールバック型 API と呼ばれることがあります。以降の記述でも、この呼称に従います。

### API ドキュメント

#### API 利用全体の流れ

libmimixfe が提供するクラスには、大きく２つの分類が有ります。第一に、音声信号処理の内容を設定するための各種設定用データクラス（構造体）であり、各信号処理機能に対応して、複数の設定用データクラスがあります。これらは、`XFETypedef.h` に定義されています。第二に、録音・信号処理を実際に実行するためのクラスで、`XFERecorder` クラスです。これは `XFERecorder.h` に定義されています。
設定用データクラスを引数に与えた `XFERecorder` クラスによって録音と信号処理を行う、というのが libmimixfe の基本的な利用方法です。具体的には、以下のような手順となります。

1. libmimixfe の各種設定用データクラスを構築し、必要に応じてメンバー変数を変更する
2. 前手順で構築したデータクラスと、ユーザー定義コールバック関数を引数に与えて `XFERecorder` クラスをコンストラクトする
3. `XFERecorder::start()` 関数の呼び出しによって、録音及び信号処理を開始する
4. メインスレッドを待機させる
5. （libmimixfe によって、音声が取得される都度、コンストラクタで指定されたユーザー定義コールバック関数が呼びされます）

#### XFERecorder クラス

録音及び信号処理を実行するクラスです。`XFERecorder.h` で定義されています。

##### コンストラクタ

``````````.cpp
XFERecorder::XFERecorder(
	const XFESourceConfig& sourceConfig,
	const XFEECConfig& ecConfig,
	const XFEVADConfig& vadConfig,
	const XFEBeamformerConfig& bfConfig,
	const XFELocalizerConfig& locConfig,
	recorderCallback_t recorderCallback,
	void *userdata);
``````````

コンストラクタでは、各種設定データクラスを引数に取ります。設定データクラスはコンストラクタで初期値が設定されているので、必要に応じてメンバー変数を変更します。次に、`recorderCallback_t` 型のユーザー定義コールバック関数を与えます。このコールバック関数は、libmimixfe によって、音声が取得された任意のタイミングでメインスレッドとは異なるスレッドで呼び出されます（コールバック型 API）、最後に、任意のユーザー定義型データを引数に与えることができます。このデータは、コールバック関数に与えられます。コールバック関数の実装例は、サンプルプログラムを参照してください。

このコンストラクタでは、標準例外のみ送出される可能性があります。

##### start() 

``````````.cpp
void XFERecorder::start()
``````````

この関数によって、録音及び信号処理が開始されます。この関数を呼び出してから、録音が開始されるまでに、マイクサブシステムを初期化するので、任意の待ち時間が発生する場合があります。待ち時間は典型的には１秒〜数秒です。

利用例は、サンプルプログラムを参照してください。典型的には、この関数はアプリケーションの中で、例えば発話単位ごとに、何回も呼び出すような用い方はしません。アプリケーションの初期化時に一度だけ呼び出し、libmimixfe は常時動作状態となるように利用します。

また `start()` された状態で、`stop()` せずに、連続的に繰り返し `start()` を呼ぶことはできません。この関数では、libmimixfe から `XFERecorderError` 例外が送出される可能性があります。

##### stop() 

``````````.cpp
int XFERecorder::stop()
``````````

この関数によって、録音及び信号処理が終了します。この関数を呼び出してから、実際に全内部処理が終了するまでには、バッファをフラッシュするため、任意の待ち時間が発生する場合があります。待ち時間は典型的には1秒〜数秒です。

この関数は、`XFERecorder` クラスのデストラクタで呼び出されるので、典型的には、ユーザープログラム側から明示的に呼び出す必要はありません。Tumbler での録音を完全に停止したい場合、例えば、マイクミュートがユーザーから指示された場合、などのみに利用します。マイクミュートは、単にコールバック関数に与えられた音声データを無視するという方法によっても実現することができますが、録音そのものを停止したい場合には `stop()` を呼ぶことになります。

返り値は、終了ステータスです。正常終了の場合 0 が返されます。

このクラスからは、標準例外のみ送出される可能性があります。

##### setLogLevel()

``````````.cpp
void XFERecorder::setLogLevel(int mask)
``````````

libmimixfe がシステムログに出力するログレベルを調整できます。`mask` はログマスクであり、LOG_UPTO マクロを利用することで簡単に設定することができます。デフォルトでは、INFO レベル以上のログが出力されます。開発段階では DEBUG レベル以上のログを出力しておくことが好ましいでしょう。利用例は、サンプルプログラムを参照してください。

##### isActive()

``````````.cpp
bool XFERecorder::isActive()
``````````

`XFERecorder` が `start()` された後、録音及び信号処理が継続されているかどうかを返します。例えば、コンストラクト直後は false、正常に `start()` が完了した場合には、true、`stop()` された場合には false が返されます。`start()` された後、何らかの内部エラー等によって録音及び信号処理が停止する場合があります。そのような場合には、false が返されることになります。

典型的には、ユーザープログラムのメインスレッドで `start()` された後、メインスレッドは、`isActive()` 関数を終了判定条件にした while ループによって待機状態とします。以下のような形が典型的です。

``````````.cpp
while(rec.isActive()){
	...
	sleep(1); // ビジーループを回避します。1 秒は典型的には待ち過ぎなので、適切な値を選択します。
}
``````````

#### XFERecorderError クラス

`XFERecorder` クラスのメンバ関数から送出される可能性のある libmimixfe 定義例外クラスです。本例外が発生した場合には、システムログに `errorstr()` の内容が自動的に記録されます。このクラスは `XFERecorder.h` で定義されています。

##### errorno()

``````````.cpp
const int XFERecorderError::errorno()
``````````

 エラーコードを返します。

##### errorstr()

``````````.cpp
const std::string XFERecorderError::errorstr()
``````````

可読なエラー文字列を返します。エラー文字列中には、エラーコードが含まれる場合があります。

#### recorderCallback_t 型

`XFERecorder` クラスのコンストラクタの引数に与える、ユーザー定義コールバック関数であり、`XFETypedef.h` で定義されています。

``````````.cpp
using recorderCallback_t = void (*)(
	short* buffer,
	size_t buflen,
	SpeechState state,
	int sourceId,
	StreamInfo* info,
	size_t infolen,
	void* userdata);
``````````

第一引数には、libmimixfe の出力として、libmimixfe が録音及び設定に従って信号処理をした処理済音声が与えられるバッファです。
第二引数は、第一引数に与えられたバッファの長さです。
第三引数は、`SpeechState` 型変数で、VAD が有効な場合に、`XFETypedef.h` で定義される VAD の判定状態が返されます。VAD が無効な場合には未定義です。
第四引数は、音源番号です。複数音源が同時検出される場合、最も
