# TypeScript Features Reference

この文書は TypeScript の構文・機能について、本プロジェクトでの対応方針と実装状況をまとめる。TypeScript 仕様は [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html) を正とする。

TypeScript 構文を frontend/parser slice に分割する運用は `docs/language-reference/frontend-parser-wave.md` を参照する。

## 仕様リファレンス

| 仕様 | URL | 用途 |
|---|---|---|
| TypeScript Handbook | <https://www.typescriptlang.org/docs/handbook/intro.html> | 公式ハンドブック |
| TypeScript Language Specification | <https://github.com/microsoft/TypeScript/blob/main/doc/spec-ARCHITECTURE.md> | 言語仕様 |
| TypeScript parser local source | `reference/typescript/src/compiler/scanner.ts`, `reference/typescript/src/compiler/parser.ts` | lexer/parser wave の primary source |
| TypeScript Playground | <https://www.typescriptlang.org/play> | オンライン実行環境 |
| DefinitelyTyped | <https://github.com/DefinitelyTyped/DefinitelyTyped> | 型定義リポジトリ |

## TypeScript 仕様詳細

### TypeScript Compiler Architecture

| コンポーネント | 説明 | 実装関連 |
|---|---|---|
| Scanner (Lexer) | 字句解析 | ソースコード → トークン列 |
| Parser | 構文解析 | トークン列 → AST |
| Binder | シンボル結合 | AST → シンボルテーブル |
| Checker (Type Checker) | 型チェック | AST + シンボル → 型情報 |
| Emitter | コード生成 | AST → JavaScript |
| Pre-processor | モジュール解決 | import/export 解決 |
| Declaration Emitter | 宣言ファイル生成 | .d.ts ファイル生成 |

### TypeScript AST ノード

| ノード種別 | 説明 | 実装関連 |
|---|---|---|
| `SourceFile` | ソースファイル | 最上位ノード |
| `Identifier` | 識別子 | 変数名、関数名など |
| `Literal` | リテラル | 数値、文字列、boolean など |
| `ArrayLiteralExpression` | 配列リテラル | `[1, 2, 3]` |
| `ObjectLiteralExpression` | オブジェクトリテラル | `{x: 1}` |
| `FunctionDeclaration` | 関数宣言 | `function f() {}` |
| `FunctionExpression` | 関数式 | `const f = function() {}` |
| `ArrowFunction` | アロー関数 | `() => {}` |
| `ClassDeclaration` | クラス宣言 | `class C {}` |
| `ClassExpression` | クラス式 | `const C = class {}` |
| `InterfaceDeclaration` | インターフェース宣言 | `interface I {}` |
| `TypeAliasDeclaration` | 型エイリアス宣言 | `type T = ...` |
| `EnumDeclaration` | 列挙型宣言 | `enum E {}` |
| `NamespaceDeclaration` | 名前空間宣言 | `namespace N {}` |
| `ModuleDeclaration` | モジュール宣言 | `module M {}` |
| `ImportDeclaration` | import 宣言 | `import ...` |
| `ExportDeclaration` | export 宣言 | `export ...` |
| `VariableStatement` | 変数宣言 | `let x = 1` |
| `ExpressionStatement` | 式文 | `x + 1` |
| `IfStatement` | if 文 | `if (x) {}` |
| `ForStatement` | for 文 | `for (;;) {}` |
| `WhileStatement` | while 文 | `while (x) {}` |
| `DoWhileStatement` | do-while 文 | `do {} while (x)` |
| `ForInStatement` | for-in 文 | `for (x in obj) {}` |
| `ForOfStatement` | for-of 文 | `for (x of arr) {}` |
| `TryStatement` | try-catch-finally 文 | `try {} catch {}` |
| `ThrowStatement` | throw 文 | `throw e` |
| `ReturnStatement` | return 文 | `return x` |
| `BreakStatement` | break 文 | `break` |
| `ContinueStatement` | continue 文 | `continue` |
| `SwitchStatement` | switch 文 | `switch (x) {}` |
| `Block` | ブロック | `{ ... }` |
| `CallExpression` | 関数呼び出し | `f()` |
| `NewExpression` | new 式 | `new C()` |
| `PropertyAccessExpression` | プロパティアクセス | `obj.x` |
| `ElementAccessExpression` | 要素アクセス | `obj[x]` |
| `BinaryExpression` | 二項演算式 | `x + y` |
| `UnaryExpression` | 単項演算式 | `-x` |
| `ConditionalExpression` | 条件式 | `x ? y : z` |
| `TypeAssertion` | 型アサーション | `x as T` |
| `AsExpression` | as 式 | `x as T` |
| `TypeReference` | 型参照 | `T` |
| `UnionTypeNode` | Union 型 | `A \| B` |
| `IntersectionTypeNode` | Intersection 型 | `A & B` |
| `TupleTypeNode` | タプル型 | `[A, B]` |
| `ArrayTypeNode` | 配列型 | `A[]` |
| `LiteralTypeNode` | リテラル型 | `"hello"` |
| `FunctionTypeNode` | 関数型 | `(x: T) => U` |
| `ConstructorTypeNode` | コンストラクタ型 | `new (x: T) => U` |
| `TypeParameterDeclaration` | 型パラメータ宣言 | `<T>` |
| `ParameterDeclaration` | パラメータ宣言 | `(x: T)` |

### TypeScript 型システム

| 型 | 説明 | 実装関連 |
|---|---|---|
| `any` | 任意の型 | 型チェック無効化 |
| `unknown` | 型安全な any | 型チェック有効 |
| `never` | 到達不能型 | 決して発生しない値 |
| `void` | 戻り値なし | undefined |
| `undefined` | undefined 値 | undefined |
| `null` | null 値 | null |
| `boolean` | 真偽値 | true / false |
| `number` | 数値 | IEEE 754 double |
| `bigint` | 大きな整数 | 任意精度整数 |
| `string` | 文字列 | UTF-16 文字列 |
| `symbol` | シンボル | 一意の値 |
| `object` | オブジェクト | 非プリミティブ値 |
| `Array<T>` | 配列型 | T の配列 |
| `ReadonlyArray<T>` | 読み取り専用配列 | 変更不可な配列 |
| `Tuple<T, U>` | タプル型 | 固定長配列 |
| `Function` | 関数型 | 任意の関数 |
| `Promise<T>` | プロミス型 | 非同期値 |
| `Record<K, V>` | レコード型 | K をキーとする V のマップ |
| `Map<K, V>` | マップ型 | ES6 Map |
| `Set<T>` | セット型 | ES6 Set |
| `Partial<T>` | 部分型 | 全プロパティを optional に |
| `Required<T>` | 必須型 | 全プロパティを required に |
| `Readonly<T>` | 読み取り専用型 | 全プロパティを readonly に |
| `Pick<T, K>` | 抽出型 | 指定プロパティのみ抽出 |
| `Omit<T, K>` | 除外型 | 指定プロパティを除外 |
| `Exclude<T, U>` | 除外型 | U に割り当て不可能な T |
| `Extract<T, U>` | 抽出型 | U に割り当て可能な T |
| `NonNullable<T>` | 非null型 | null / undefined を除外 |
| `ReturnType<T>` | 戻り値型 | 関数の戻り値型 |
| `Parameters<T>` | パラメータ型 | 関数のパラメータ型タプル |
| `InstanceType<T>` | インスタンス型 | クラスのインスタンス型 |

### TypeScript コンパイラオプション

| オプション | 説明 | 実装関連 |
|---|---|---|
| `--strict` | 厳格モード有効化 | 全ての厳格オプション有効 |
| `--noImplicitAny` | any 暗黙禁止 | 型注釈必須 |
| `--strictNullChecks` | null/undefined 厳格チェック | null/undefined 厳密 |
| `--strictFunctionTypes` | 関数型厳格チェック | 関数型の共変/反変厳密 |
| `--strictBindCallApply` | bind/call/apply 厳格チェック | メソッド呼び出し厳密 |
| `--strictPropertyInitialization` | プロパティ初期化厳格チェック | クラスプロパティ初期化必須 |
| `--noImplicitThis` | this 暗黙禁止 | this 型注釈必須 |
| `--alwaysStrict` | 常に厳格モード | `"use strict"` 自動挿入 |
| `--noUnusedLocals` | 未使用ローカル変数エラー | 未使用変数検出 |
| `--noUnusedParameters` | 未使用パラメータエラー | 未使用パラメータ検出 |
| `--noImplicitReturns` | 暗黙リターンエラー | 全パスで return 必須 |
| `--noFallthroughCasesInSwitch` | switch fallthrough エラー | break/return 必須 |
| `--esModuleInterop` | ES Module 相互運用 | CommonJS 互換性向上 |
| `--allowSyntheticDefaultImports` | 合成デフォルトインポート有効化 | デフォルトインポート許可 |
| `--resolveJsonModule` | JSON モジュール解決 | .json ファイル import 許可 |
| `--moduleResolution` | モジュール解決戦略 | node / classic |
| `--target` | 出力 ECMAScript バージョン | ES3 / ES5 / ES2015+ |
| `--module` | モジュールシステム | CommonJS / ES6 / UMD など |
| `--lib` | 使用ライブラリ定義 | DOM / ES5 / ES2015+ など |
| `--outDir` | 出力ディレクトリ | コンパイル結果出力先 |
| `--outFile` | 出力ファイル | 単一ファイルに結合 |
| `--declaration` | 宣言ファイル生成 | .d.ts ファイル生成 |
| `--declarationMap` | 宣言マップ生成 | .d.ts.map ファイル生成 |
| `--sourceMap` | ソースマップ生成 | .map ファイル生成 |
| `--removeComments` | コメント削除 | 出力からコメント削除 |
| `--preserveConstEnums` | const enum 保存 | enum 実行時コード生成 |
| `--isolatedModules` | モジュール単位トランスパイル | 各モジュール単独で有効なコード生成 |

## 型システム

| 機能 | TypeScript | 対応方針 | 実装状況 | 優先度 | Issue ID |
|---|---|---|---|---|---|
| 基本型 (`string`, `number`, `boolean`) | ES3 | 型注釈として解析 | 未実装 | P2 | - |
| 配列型 `string[]` | ES3 | 型注釈として解析 | 未実装 | P2 | - |
| タプル型 `[string, number]` | ES3 | 型注釈として解析 | 未実装 | P2 | - |
| オブジェクト型 `{x: number}` | ES3 | 型注釈として解析 | 未実装 | P2 | - |
| Union 型 `string \| number` | ES3 | 型注釈として解析 | 未実装 | P2 | - |
| Intersection 型 `A & B` | ES3 | 型注釈として解析 | 未実装 | P2 | - |
| Literal 型 `"hello"` | ES3 | 型注釈として解析 | 未実装 | P2 | - |
| `any` | ES3 | 型チェック無効化 | 未実装 | P2 | - |
| `unknown` | ES3 | 型安全な any | 未実装 | P2 | - |
| `never` | ES3 | 到達不能型 | 未実装 | P2 | - |
| `void` | ES3 | 戻り値なし | 未実装 | P2 | - |

## 型アノテーション

| 機能 | TypeScript | 対応方針 | 実装状況 | 優先度 | Issue ID |
|---|---|---|---|---|---|
| 変数アノテーション `let x: number` | ES3 | 型情報を解析 | 未実装 | P2 | - |
| 関数パラメータ `fn(x: number)` | ES3 | 型情報を解析 | 未実装 | P2 | - |
| 戻り値型 `fn(): number` | ES3 | 型情報を解析 | 未実装 | P2 | - |
| アロー関数 `(x: number): number => x` | ES3 | 型情報を解析 | 未実装 | P2 | - |
| オブジェクトリテラル型 `{x: number}` | ES3 | 型情報を解析 | 未実装 | P2 | - |

## インターフェースと型エイリアス

| 機能 | TypeScript | 対応方針 | 実装状況 | 優先度 | Issue ID |
|---|---|---|---|---|---|
| `interface` | ES3 | type-only parse | 未実装 | P2 | - |
| `type` alias | ES3 | type-only parse | 未実装 | P2 | - |
| 継承 `interface A extends B` | ES3 | type-only parse | 未実装 | P2 | - |
| 宣言マージ | ES3 | type-only parse | 未実装 | P2 | - |

## ジェネリクス

| 機能 | TypeScript | 対応方針 | 実装状況 | 優先度 | Issue ID |
|---|---|---|---|---|---|
| ジェネリック関数 `fn<T>(x: T): T` | ES3 | erased type syntax | 未実装 | P2 | - |
| ジェネリッククラス `class C<T>` | ES3 | erased type syntax | 未実装 | P2 | - |
| ジェネリック型エイリアス `type Pair<T> = [T, T]` | ES3 | erased type syntax | 未実装 | P2 | - |
| 制約 `T extends U` | ES3 | 型情報による最適化 | 未実装 | P2 | - |
| デフォルト型引数 `T = string` | ES3 | erased type syntax | 未実装 | P2 | - |

## 列挙型

| 機能 | TypeScript | 対応方針 | 実装状況 | 優先度 | Issue ID |
|---|---|---|---|---|---|
| numeric enum | ES3 | numeric enum subset | 未実装 | P2 | - |
| string enum | ES3 | string enum | 未実装 | P2 | - |
| const enum | ES3 | const enum | 未実装 | P2 | - |
| enum メンバーアクセス | ES3 | プロパティアクセス | 未実装 | P2 | - |

## 名前空間

| 機能 | TypeScript | 対応方針 | 実装状況 | 優先度 | Issue ID |
|---|---|---|---|---|---|
| `namespace` | ES3 | 未実装 (unsupported-namespace) | 未実装 | P3 | - |
| `module` (namespace alias) | ES3 | 未実装 | 未実装 | P3 | - |
| 宣言マージ | ES3 | 未実装 | 未実装 | P3 | - |

## デコレータ

| 機能 | TypeScript | 対応方針 | 実装状況 | 優先度 | Issue ID |
|---|---|---|---|---|---|
| クラスデコレータ | ES5 | 未実装 | 未実装 | P3 | - |
| メソッドデコレータ | ES5 | 未実装 | 未実装 | P3 | - |
| アクセサデコレータ | ES5 | 未実装 | 未実装 | P3 | - |
| プロパティデコレータ | ES5 | 未実装 | 未実装 | P3 | - |
| パラメータデコレータ | ES5 | 未実装 | 未実装 | P3 | - |

## 高度な型

| 機能 | TypeScript | 対応方針 | 実装状況 | 優先度 | Issue ID |
|---|---|---|---|---|---|
| Conditional types `T extends U ? X : Y` | ES3 | 型情報による最適化 | 未実装 | P2 | - |
| Mapped types `{ [K in keyof T]: U }` | ES3 | 型情報による最適化 | 未実装 | P2 | - |
| Keyof type `keyof T` | ES3 | 型情報による最適化 | 未実装 | P2 | - |
| Infer type `infer R` | ES3 | 型情報による最適化 | 未実装 | P2 | - |
| Template literal types `` `hello${T}` `` | ES3 | 型情報による最適化 | 未実装 | P2 | - |
| Utility types (`Partial`, `Required`, etc.) | ES3 | 型情報による最適化 | 未実装 | P2 | - |

## 型アサーションとキャスト

| 機能 | TypeScript | 対応方針 | 実装状況 | 優先度 | Issue ID |
|---|---|---|---|---|---|
| 型アサーション `x as T` | ES3 | 型チェック無効化 | 未実装 | P2 | - |
| 角括弧キャスト `<T>x` | ES3 | 型チェック無効化 | 未実装 | P2 | - |
| const assertion `x as const` | ES3 | リテラル型推論 | 未実装 | P2 | - |
| Non-null assertion `x!` | ES3 | null/undefined チェック無効化 | 未実装 | P2 | - |

## 型ガード

| 機能 | TypeScript | 対応方針 | 実装状況 | 優先度 | Issue ID |
|---|---|---|---|---|---|
| `typeof` ガード `typeof x === "string"` | ES3 | 型 narrowing | 未実装 | P2 | - |
| `instanceof` ガード `x instanceof C` | ES3 | 型 narrowing | 未実装 | P2 | - |
| カスタム型ガード `x is T` | ES3 | 型 narrowing | 未実装 | P2 | - |
| 判別可能ユニオン | ES3 | 型 narrowing | 未実装 | P2 | - |

## モジュール

| 機能 | TypeScript | 対応方針 | 実装状況 | 優先度 | Issue ID |
|---|---|---|---|---|---|
| `import` / `export` | ES6 | single file only → relative static import/export | 未実装 | P1 | - |
| `import type` | ES6 | type-only import | 未実装 | P2 | - |
| `export type` | ES6 | type-only export | 未実装 | P2 | - |
| `import = require()` | CommonJS | compile-time builtin resolution | 未実装 | P2 | - |
| `export =` | CommonJS | CommonJS export | 未実装 | P2 | - |
| `declare module` | ES3 | ambient module | 未実装 | P2 | - |

## その他

| 機能 | TypeScript | 対応方針 | 実装状況 | 優先度 | Issue ID |
|---|---|---|---|---|---|
| 型推論 | ES3 | 型情報を推論 | 未実装 | P2 | - |
| 型エラー診断 | ES3 | 型エラーを報告 | 未実装 | P2 | - |
| `strict` モード | ES3 | 厳密な型チェック | 未実装 | P2 | - |
| `--noImplicitAny` | ES3 | any 暗黙禁止 | 未実装 | P2 | - |
| `--strictNullChecks` | ES3 | null/undefined 厳密チェック | 未実装 | P2 | - |
| `--strictFunctionTypes` | ES3 | 関数型厳密チェック | 未実装 | P2 | - |
| Triple-slash directives | ES3 | コンパイラ指示 | 未実装 | P2 | - |
| `tsconfig.json` | ES3 | コンパイラ設定 | 未実装 | P2 | - |

## Ambient Declarations

TypeScript の `declare` キーワードと ambient declaration ファミリーは、実行時に影響を与えない宣言のみを erasure し、runtime に影響を与える形式は明示的に拒否する。

### Classification

| Category | Ambient form | Erasure behavior | Diagnostic |
|---|---|---|---|
| A. Erased (no runtime effect) | `declare function name(...): T;` | Empty function emitted for name resolution; no callable JS runtime binding | Pass |
| A. Erased | `declare const/let/var name: T;` | Fully erased; no variable binding in lowered IR | Pass |
| A. Erased | `declare class Name { ... }` | Entire class body erased; no prototype/constructor emitted | Pass |
| A. Erased | `declare enum Name { ... }` | Enum body erased; no reverse mapping or runtime object | Pass |
| A. Erased | `declare interface Name { ... }` | Interface erased within declare block | Pass |
| A. Erased | `declare type T = ...` | Type alias erased within declare block | Pass |
| A. Erased | `namespace/Module Name { ... }` (no `declare` needed) | Entire namespace body erased; routes to `UnsupportedModule` | `UnsupportedModule` |
| A. Erased | `declare namespace/module Name { ... }` | Same as above; `declare` is optional for namespaces | `UnsupportedModule` |
| A. Erased | Class element `declare prop: T;` | Field declaration erased; no instance slot created | Pass |
| B. Rejected (runtime impact) | `declare global { ... }` | Rejected; cannot be safely erased | `UnsupportedTypeScriptSyntax` |
| B. Rejected | `declare const x = value;` (with initializer) | Rejected; initializer would create runtime binding | `UnsupportedTypeScriptSyntax` |
| B. Rejected | Ambient class element `declare prop = value;` | Rejected; initializer would create runtime slot | `UnsupportedTypeScriptSyntax` |
| C. Parsed for scope only | `declare function name(...)` | Empty function emitted in pending_statements so the name is registered in scope | Pass (scope registration) |

### Erasure scope summary

The ambient erasure boundary covers the intersection of `docs/05-compatibility-and-semantics.md` category 4 (reject or erase precisely) for `ambient-declaration` labeled cases. The following contracts apply:

1. **No silent runtime binding**: Erased ambient forms must not introduce runtime objects, functions, classes, enums, or variable bindings.
2. **Scope name registration**: `declare function` emits an empty function body so callers can reference the name without unresolved-reference diagnostics.
3. **Module-shaped erasure**: `declare module` / `declare namespace` erases the body and routes to `UnsupportedModule` when module-shape handling is the remaining blocker.
4. **Initializer rejection**: Any ambient declaration with a value initializer (`= expr`) is rejected with a diagnostic, regardless of the expression type.

### Covered ambient issues

The following generated ambient-declaration issues from `tsc` and `tsgo` coverage are covered by this erasure boundary (issue 400 + current implementation):

| Issue ID | Title | Erasure category |
|---|---|---|
| 140 | ambientClassDeclarationWithExtends (tsc) | A. Erased (class) |
| 145 | ambientEnum (tsc) | A. Erased (enum) |
| 150 | ambientExternalModuleReopen (tsc) | A. Erased (module/namespace) |
| 160 | ambientModules (tsc) | A. Erased (module/namespace) |
| 162 | ambientPropertyDeclarationInJs (tsc) | A. Erased (class element) |
| 144 | ambientConstLiterals (tsc) | A. Erased (variable) |
| 148 | ambientExportDefaultErrors (tsc) | A. Erased (export declare) |
| 519 | ambientClassDeclaredBeforeBase | A. Erased (class) |
| 522 | ambientErrors | A. Erased (general) |
| 531 | ambientModuleExports | A. Erased (module) |
| 534 | ambientModules | A. Erased (module) |
| 536 | ambientRequireFunction | A. Erased (variable) |
| 607 | ambientEnumElementInitializer | A. Erased (enum) |
| 608 | ambientErrors | A. Erased (general) |

Issues that remain open because they require more than erasure behavior (e.g., overload merging, declaration emit, runtime semantics) are not covered by this boundary and are tracked individually.

## 実装方針の原則

1. **TypeScript 準拠**: 可能な限り TypeScript 仕様に準拠する
2. **型情報活用**: 型情報を最適化と診断に活用する
3. **実行時消去**: 型は実行時に消去される（実行時検査は行わない）
4. **AssemblyScript 固有機能**: `i32`、`i64`、`usize` などは入力言語として扱わない
5. **段階的実装**: 型解析 → 型チェック → 最適化の順で実装する
