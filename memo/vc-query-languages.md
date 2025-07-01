# Verifiable Credentialsにおけるクエリ言語の比較：DCQLとPresentation Exchange

Verifiable Credentials (VCs) のエコシステムにおいて、検証者が提示者から特定のCredentialを要求したり、提示者がCredentialを提示する際に、その内容を記述するためのクエリ言語がいくつか存在します。ここでは、主要な2つのクエリ言語、Digital Credential Query Language (DCQL) と Presentation Exchange (PE) について、その特徴と使用法を比較し、具体的な構文例とVCの例を交えながら詳細に解説します。

## 0. Verifiable Credential (VC) の例

DCQLやPresentation Exchangeのクエリを理解するために、まず対象となるVerifiable Credentialの基本的な構造をいくつか示します。これらのVCは、JSON-LD形式で表現されます。

### 運転免許証 (DriversLicense)

```json
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "https://www.w3.org/2018/credentials/examples/v1"
  ],
  "id": "http://example.edu/credentials/123",
  "type": [
    "VerifiableCredential",
    "DriversLicense"
  ],
  "issuer": "did:example:123",
  "issuanceDate": "2023-01-01T12:00:00Z",
  "credentialSubject": {
    "id": "did:example:456",
    "name": "John Doe",
    "dateOfBirth": "2000-05-15",
    "age": 23,
    "licenseNumber": "DL123456789"
  }
}
```

### 大学の学位 (UniversityDegreeCredential)

```json
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "https://www.w3.org/2018/credentials/examples/v1"
  ],
  "id": "http://example.edu/credentials/degree/888",
  "type": [
    "VerifiableCredential",
    "UniversityDegreeCredential"
  ],
  "issuer": "did:example:university",
  "issuanceDate": "2022-03-01T10:00:00Z",
  "credentialSubject": {
    "id": "did:example:student789",
    "name": "Jane Smith",
    "degree": {
      "type": "BachelorDegree",
      "name": "Computer Science"
    },
    "university": "Example University"
  }
}
```

### パスポート (Passport)

```json
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "https://www.w3.org/2018/credentials/examples/v1"
  ],
  "id": "http://example.com/credentials/passport/abc",
  "type": [
    "VerifiableCredential",
    "Passport"
  ],
  "issuer": "did:example:government",
  "issuanceDate": "2020-07-01T09:00:00Z",
  "credentialSubject": {
    "id": "did:example:citizen101",
    "name": "Alice Wonderland",
    "nationality": "Wonderland",
    "passportNumber": "P1234567"
  }
}
```

## 1. Digital Credential Query Language (DCQL)

### 概要
Digital Credential Query Language (DCQL) は、OpenID for Verifiable Presentations (OpenID4VP) の仕様の一部として定義されている、JSONベースのクエリ言語です。検証者が提示者に対して、どのようなデジタルCredentialを求めているかを記述するために使用されます。

### 特徴
- **JSONベース**: クエリはJSON形式で記述され、人間が読みやすく、機械が処理しやすい構造です。
- **Credentialの特定**: Credentialのフォーマット、タイプ、特定のフィールド（属性）など、詳細な条件を指定してCredentialを要求できます。
- **柔軟なクエリ**: 論理演算子（AND, ORなど）を用いて、複数の条件を組み合わせた複雑なクエリを作成できます。
- **OpenID4VPとの連携**: OpenID4VPフローの中で、検証者が提示者にCredentialの提示を要求する際に利用されます。

### 構文と使用例

DCQLは、`credential_query`というJSONオブジェクトで表現されます。このオブジェクトは、要求するCredentialの条件を記述するための様々なプロパティを持ちます。

#### 基本的なクエリ

特定の`type`を持つCredentialを要求する例：

```json
{
  "credential_query": [
    {
      "type": "VerifiableCredential",
      "credential_type": "UniversityDegreeCredential"
    }
  ]
}
```

**上記のクエリが対象とするVCの例:**

- 大学の学位 (UniversityDegreeCredential)

#### 複数の条件の指定

`credential_type`と`format`の両方を指定する例：

```json
{
  "credential_query": [
    {
      "type": "VerifiableCredential",
      "credential_type": "DriversLicense",
      "format": "jwt_vc_json"
    }
  ]
}
```

**上記のクエリが対象とするVCの例:**

- 運転免許証 (DriversLicense) (ただし、`jwt_vc_json`フォーマットである必要があります)

#### 属性に基づくフィルタリング

Credentialの特定の属性（クレーム）に基づいてフィルタリングを行う場合、`fields`プロパティを使用します。`fields`は、CredentialのJSON構造内のパスと、そのパスに期待される値を指定します。

例えば、`age`が`18`以上である運転免許証を要求する例：

```json
{
  "credential_query": [
    {
      "type": "VerifiableCredential",
      "credential_type": "DriversLicense",
      "fields": [
        {
          "path": [
            "$.vc.credentialSubject.age"
          ],
          "filter": {
            "type": "number",
            "minimum": 18
          }
        }
      ]
    }
  ]
}
```

**上記のクエリが対象とするVCの例:**

- 運転免許証 (DriversLicense) で、`credentialSubject.age`が18以上のもの。

#### 論理演算子

複数の条件を組み合わせるために、`credential_query`配列内に複数のオブジェクトを記述することでAND条件を表現できます。また、`credential_query`の各要素はOR条件として扱われます。

例えば、「運転免許証」または「パスポート」のいずれかを要求する例：

```json
{
  "credential_query": [
    {
      "type": "VerifiableCredential",
      "credential_type": "DriversLicense"
    },
    {
      "type": "VerifiableCredential",
      "credential_type": "Passport"
    }
  ]
}
```

**上記のクエリが対象とするVCの例:**

- 運転免許証 (DriversLicense) または パスポート (Passport) のいずれか。

### DCQL-TS

`https://tac.openwallet.foundation/projects/dcql-ts/` で言及されているDCQL-TSは、DCQL仕様のTypeScriptリファレンス実装です。これは、DCQLクエリの構築、解析、検証、およびCredentialがクエリに合致するかどうかを評価するためのエンジンを提供します。これにより、開発者はDCQLを容易にアプリケーションに統合できます。

## 2. Presentation Exchange (PE)

### 概要
Presentation Exchange (PE) は、Decentralized Identity Foundation (DIF) によって開発された仕様で、検証者が提示者から証明（Proof）を要求し、提示者がその証明を提出する方法を定義します。特定のCredentialフォーマットや通信プロトコルに依存しない、汎用的なフレームワークを提供します。

### 特徴
- **Presentation Definition**: 検証者が要求する証明の要件を記述するためのデータフォーマットです。`input_descriptors` を用いて、必要な情報の種類や形式、制約などを詳細に定義します。
- **Presentation Submission**: 提示者が提出する証明の内容を記述するためのデータフォーマットです。どのCredentialのどの部分が、要求されたどの`input_descriptor`に対応するかを示します。
- **フォーマット非依存**: JWTs、Verifiable Credentialsなど、特定のCredentialフォーマットに限定されません。
- **プロトコル非依存**: OpenID Connect、DIDCommなど、特定の通信プロトコルに依存しません。
- **柔軟な要件定義**: `submission_requirements` を用いて、複数の`input_descriptors`間の論理的な関係（例：AND, OR, XOR）や、提示するCredentialの最小/最大数などを定義できます。

### 構文と使用例

PEの主要な構成要素は`Presentation Definition`と`Presentation Submission`です。

#### Presentation Definition

検証者が提示者に要求する証明の要件を定義します。

```json
{
  "id": "32f54163-7166-48f1-93d8-ff217bdb0653",
  "input_descriptors": [
    {
      "id": "drivers_license_input",
      "name": "Drivers License",
      "purpose": "Please present your valid Drivers License.",
      "constraints": {
        "fields": [
          {
            "path": [
              "$.vc.type"
            ],
            "filter": {
              "type": "string",
              "pattern": "DriversLicense"
            }
          },
          {
            "path": [
              "$.vc.credentialSubject.age"
            ],
            "filter": {
              "type": "number",
              "minimum": 18
            }
          }
        ]
      }
    }
  ]
}
```

**上記のPresentation Definitionが対象とするVCの例:**

- 運転免許証 (DriversLicense) で、`type`が`DriversLicense`であり、かつ`credentialSubject.age`が18以上のもの。

#### Submission Requirements

複数の`input_descriptors`間の論理的な関係や、提示するCredentialの最小/最大数などを定義します。

例えば、「運転免許証」と「パスポート」のいずれか一方を要求する例：

```json
{
  "id": "example-presentation-definition",
  "input_descriptors": [
    {
      "id": "drivers_license_input",
      "name": "Drivers License",
      "purpose": "Please present your valid Drivers License.",
      "constraints": {
        "fields": [
          {
            "path": [
              "$.vc.type"
            ],
            "filter": {
              "type": "string",
              "pattern": "DriversLicense"
            }
          }
        ]
      }
    },
    {
      "id": "passport_input",
      "name": "Passport",
      "purpose": "Please present your valid Passport.",
      "constraints": {
        "fields": [
          {
            "path": [
              "$.vc.type"
            ],
            "filter": {
              "type": "string",
              "pattern": "Passport"
            }
          }
        ]
      }
    }
  ],
  "submission_requirements": [
    {
      "rule": "pick",
      "count": 1,
      "from": "all",
      "input_descriptors": [
        "drivers_license_input",
        "passport_input"
      ]
    }
  ]
}
```

**上記のPresentation Definitionが対象とするVCの例:**

- 運転免許証 (DriversLicense) または パスポート (Passport) のいずれか1つ。

#### Presentation Submission

提示者がPresentation Definitionに基づいてCredentialを提示する際に作成するデータです。どのCredentialのどの部分が、要求されたどの`input_descriptor`に対応するかを示します。

```json
{
  "id": "a30e743e-7d6e-4f23-8e3c-21b72f21543e",
  "definition_id": "32f54163-7166-48f1-93d8-ff217bdb0653",
  "descriptor_map": [
    {
      "id": "drivers_license_input",
      "format": "jwt_vc_json",
      "path": "$",
      "path_nested": {
        "format": "jwt_vc_json",
        "path": "$.verifiableCredential[0]"
      }
    }
  ]
}
```

**上記のPresentation Submissionが示すVCの例:**

- 提示者が、`definition_id`が`32f54163-7166-48f1-93d8-ff217bdb0653`のPresentation Definitionに対して、`drivers_license_input`に対応する`jwt_vc_json`形式のCredentialを提示したことを示しています。

## 3. DCQLとPEの比較

| 特徴             | Digital Credential Query Language (DCQL)                               | Presentation Exchange (PE)                                         |
| :--------------- | :--------------------------------------------------------------------- | :----------------------------------------------------------------- |
| **目的**         | 検証者が特定のデジタルCredentialを要求する                               | 検証者が証明（Proof）を要求し、提示者がそれを提出する方法を定義する |
| **基盤**         | OpenID for Verifiable Presentations (OpenID4VP) の一部                   | Decentralized Identity Foundation (DIF) の独立した仕様             |
| **フォーマット** | JSONベース                                                             | JSONベース                                                         |
| **対象**         | 主にVerifiable Credentials                                             | 特定のCredentialフォーマットに非依存（汎用的）                     |
| **柔軟性**       | Credentialのフォーマット、タイプ、フィールドに基づく詳細なクエリ         | `input_descriptors`と`submission_requirements`による柔軟な要件定義 |\
| **連携**         | OpenID4VPフロー内で利用                                                | 任意の通信プロトコルと連携可能                                     |
| **構文の複雑さ** | 比較的シンプル。Credentialの属性に直接クエリを記述。                     | `Presentation Definition`と`Presentation Submission`の2つの主要な構造を持ち、より複雑な論理を表現可能。 |
| **ユースケース** | 特定のCredentialの属性に基づくフィルタリング、単一のCredentialの要求。   | 複数のCredentialの組み合わせ、部分的な開示、複雑な論理条件に基づく証明の要求。 |

## 結論

DCQLとPEは、Verifiable Credentialsのエコシステムにおいて、検証者と提示者の間でCredentialや証明のやり取りを円滑にするための重要なツールです。

- **DCQL** は、OpenID4VPの文脈で、特定のCredentialの属性に基づいて詳細なフィルタリングや選択を行いたい場合に特に有用です。Credentialの具体的な内容に焦点を当てたクエリに適しており、比較的シンプルな構文でCredentialの要求を記述できます。

- **Presentation Exchange** は、より汎用的なフレームワークであり、Credentialのフォーマットや通信プロトコルに依存せず、複雑な証明の組み合わせや部分的な開示（Selective Disclosure）を柔軟に定義したい場合に適しています。複数のCredentialを組み合わせて提示するシナリオや、プライバシーを重視した情報開示の制御に強みがあり、より高度な要件定義が可能です。

両者は異なる目的とスコープを持っていますが、相互に補完し合う関係にあります。特定のユースケースにおいては、両方の仕様の概念を組み合わせて利用することも考えられます。例えば、PEの`Presentation Definition`内で、特定の`input_descriptor`の`constraints`としてDCQLのクエリを使用するといった応用も理論的には可能です。これにより、PEの柔軟なフレームワークの中で、DCQLの詳細なCredential属性フィルタリング能力を活用できます。

これらのクエリ言語を理解し適切に活用することで、Verifiable Credentialsの相互運用性と実用性が向上し、よりセキュアでプライバシーに配慮したデジタルアイデンティティの実現に貢献します。
