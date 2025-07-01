# Verifiable Credentialsにおけるクエリ言語の比較：DCQLとPresentation Exchange

本稿では、Verifiable Credentials (VCs) エコシステムにおける主要な2つのクエリ言語、Digital Credential Query Language (DCQL) と Presentation Exchange (PE) を比較検討する。両者の特徴、構文、およびVCの例を提示し、それぞれの設計思想と適用範囲を考察する。

## 0. Verifiable Credential (VC) の例

DCQLおよびPEのクエリ対象となるVerifiable Credentialの構造を以下に示す。これらはJSON-LD形式で表現される。

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
DCQLは、OpenID for Verifiable Presentations (OpenID4VP) の一部として定義されるJSONベースのクエリ言語である。検証者が提示者に対し、特定のデジタルCredentialの要求を記述するために用いられる。DCQLはW3C Verifiable Credentialsの構造（`type`や`credentialSubject`内のデータ）と互換性を有する。

### 特徴
- **JSONベース**: クエリはJSON形式で記述され、可読性および機械処理性に優れる。
- **Credentialの特定**: Credentialのフォーマット、タイプ、特定のフィールド（属性）など、詳細な条件指定が可能である。
- **柔軟なクエリ**: 論理演算子（AND, OR）を用いた複雑なクエリの構築が可能である。
- **OpenID4VPとの連携**: OpenID4VPフローにおけるCredential提示要求に利用される。

### 構文と使用例

DCQLは`credential_query`オブジェクトとして表現され、要求するCredentialの条件を記述する。

#### 基本的なクエリ

特定の`type`を持つCredentialの要求例：

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

**対象VC例:** 大学の学位 (UniversityDegreeCredential)

#### 複数の条件の指定

`credential_type`と`format`の同時指定例：

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

**対象VC例:** 運転免許証 (DriversLicense) (ただし、`jwt_vc_json`フォーマット)

#### 属性に基づくフィルタリング

Credentialの特定の属性（クレーム）に基づくフィルタリングには`fields`プロパティを用いる。`fields`は、CredentialのJSON構造内のパスと、そのパスに期待される値を指定する。

例：`age`が`18`以上である運転免許証の要求：

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

**対象VC例:** 運転免許証 (DriversLicense) で、`credentialSubject.age`が18以上のもの。

#### 論理演算子

`credential_query`配列内の複数のオブジェクトはAND条件として機能し、各要素はOR条件として扱われる。

例：「運転免許証」または「パスポート」のいずれかの要求：

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

**対象VC例:** 運転免許証 (DriversLicense) または パスポート (Passport) のいずれか。

### DCQL-TS

`https://tac.openwallet.foundation/projects/dcql-ts/` にて言及されるDCQL-TSは、DCQL仕様のTypeScriptリファレンス実装である。これは、DCQLクエリの構築、解析、検証、およびCredentialのクエリ合致評価エンジンを提供する。

## 2. Presentation Exchange (PE)

### 概要
PEは、Decentralized Identity Foundation (DIF) が開発した仕様であり、検証者による証明（Proof）要求と提示者による提出方法を定義する。特定のCredentialフォーマットや通信プロトコルに依存しない汎用的なフレームワークを提供する。

### 特徴
- **Presentation Definition**: 検証者が要求する証明の要件を記述するデータフォーマット。`input_descriptors`により、必要な情報の種類、形式、制約を詳細に定義する。
- **Presentation Submission**: 提示者が提出する証明の内容を記述するデータフォーマット。提示されたCredentialと、要求された`input_descriptor`のマッピングを示す。
- **フォーマット非依存**: JWTs、Verifiable Credentialsなど、特定のCredentialフォーマットに限定されない。
- **プロトコル非依存**: OpenID Connect、DIDCommなど、特定の通信プロトコルに依存しない。
- **柔軟な要件定義**: `submission_requirements`により、複数の`input_descriptors`間の論理関係（AND, OR, XOR）や、提示するCredentialの最小/最大数を定義可能である。

### 構文と使用例

PEの主要構成要素は`Presentation Definition`と`Presentation Submission`である。

#### Presentation Definition

検証者が提示者に要求する証明の要件を定義する。

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

**対象Presentation Definitionが対象とするVC例:** 運転免許証 (DriversLicense) で、`type`が`DriversLicense`であり、かつ`credentialSubject.age`が18以上のもの。

#### Submission Requirements

複数の`input_descriptors`間の論理関係や、提示するCredentialの最小/最大数を定義する。

例：「運転免許証」と「パスポート」のいずれか一方の要求：

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

**対象Presentation Definitionが対象とするVC例:** 運転免許証 (DriversLicense) または パスポート (Passport) のいずれか1つ。

#### Presentation Submission

提示者がPresentation Definitionに基づきCredentialを提示する際に作成するデータ。提示されたCredentialと、要求された`input_descriptor`のマッピングを示す。

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

**対象Presentation Submissionが示すVC例:** 提示者が、`definition_id`が`32f54163-7166-48f1-93d8-ff217bdb0653`のPresentation Definitionに対し、`drivers_license_input`に対応する`jwt_vc_json`形式のCredentialを提示したことを示す。

## 3. DCQLとPEの比較

| 特徴             | Digital Credential Query Language (DCQL)                               | Presentation Exchange (PE)                                         |
| :--------------- | :--------------------------------------------------------------------- | :----------------------------------------------------------------- |
| **目的**         | 特定のデジタルCredentialの要求                                         | 証明（Proof）要求と提出方法の定義                                  |
| **基盤**         | OpenID for Verifiable Presentations (OpenID4VP) の一部                   | Decentralized Identity Foundation (DIF) の独立した仕様             |
| **フォーマット** | JSONベース                                                             | JSONベース                                                         |
| **対象**         | 主にVerifiable Credentials                                             | 特定のCredentialフォーマットに非依存（汎用的）                     |
| **柔軟性**       | Credentialのフォーマット、タイプ、フィールドに基づく詳細なクエリ         | `input_descriptors`と`submission_requirements`による柔軟な要件定義 |
| **連携**         | OpenID4VPフロー内で利用                                                | 任意の通信プロトコルと連携可能                                     |
| **構文の複雑さ** | 比較的シンプル。Credential属性に直接クエリを記述。                     | `Presentation Definition`と`Presentation Submission`の2つの主要構造を持ち、より複雑な論理を表現可能。 |
| **ユースケース** | 特定のCredential属性に基づくフィルタリング、単一Credentialの要求。       | 複数Credentialの組み合わせ、部分開示、複雑な論理条件に基づく証明要求。 |

## 4. DCQLの簡潔さとPEの複雑性、およびPEの利点

### DCQLの簡潔性

DCQLは、**単一のVerifiable Credentialの属性フィルタリングと特定Credentialの要求**に特化している。この目的の限定性により、クエリ構造は直感的かつフラットであり、Credentialの`type`や`credentialSubject`内の特定フィールドへの直接アクセスを可能にする。これは、検証者が「このCredentialが欲しい」という意図を直接的に表現するのに適している。

### PEの複雑性

PEは、DCQLよりも広範なシナリオに対応するため、その設計は複雑である。主な要因は以下の通りである。

1.  **複数Credentialの組み合わせ:** PEは、単一Credentialに留まらず、**複数の異なるCredentialを組み合わせて提示を要求**できる。例として、「運転免許証」と「住所証明」の同時要求が挙げられる。
2.  **部分開示 (Selective Disclosure):** 提示者がCredentialの全情報を開示せず、検証者が真に必要とする情報のみを提示するメカニズムを提供する。これによりプライバシー保護が強化されるが、情報開示の細やかな制御には複雑な構造（`input_descriptors`内の`fields`や`path`）が不可欠となる。
3.  **複雑な論理関係の定義:** `submission_requirements`を通じて、提示されるCredential間の複雑な論理関係（AND, OR, XORなど）や、提示Credentialの数（最小/最大）を定義可能である。これにより、高度な柔軟性が得られる反面、記述の複雑性が増大する。
4.  **Presentation DefinitionとPresentation Submissionの分離:** 検証者による要求定義（`Presentation Definition`）と、提示者による応答（`Presentation Submission`）が明確に分離されている。これはプロトコルレベルの柔軟性を高めるが、概念理解と実装の複雑性を伴う。

### PEの利点

PEの複雑性は、その強力な機能と柔軟性に由来する。主な利点は以下の通りである。

1.  **高度なプライバシー保護:** Selective Disclosureにより、提示者は検証者が真に必要とする情報のみを開示し、それ以外の情報を秘匿できる。これは個人情報保護の観点から極めて重要である。
2.  **柔軟な提示要件の定義:** 複数Credentialの組み合わせ要求や、特定Credentialの特定属性のみの提示要求など、複雑なシナリオに柔軟に対応可能である。これにより、多様なユースケースに対応したきめ細やかな検証プロセスを構築できる。
3.  **フォーマット・プロトコル非依存:** 特定のCredentialフォーマット（W3C VC以外も含む）や通信プロトコルに縛られず、汎用的に利用可能である。これにより、異なるエコシステム間での相互運用性と将来的な拡張性が確保される。
4.  **提示の検証可能性:** `Presentation Submission`を通じて、提示されたCredentialが`Presentation Definition`の要件を正確に満たしているかを検証者が容易に確認できる。これにより、検証プロセスの信頼性が向上する。

## 結論

DCQLとPEは、VCエコシステムにおいて、検証者と提示者間のCredentialおよび証明の円滑なやり取りを促進する重要なツールである。

- **DCQL** は、OpenID4VPの文脈において、特定のCredential属性に基づく詳細なフィルタリングや選択に有用である。Credentialの具体的な内容に焦点を当てたクエリに適しており、比較的シンプルな構文でCredential要求を記述できる。

- **Presentation Exchange** は、より汎用的なフレームワークであり、Credentialフォーマットや通信プロトコルに依存せず、複雑な証明の組み合わせや部分開示を柔軟に定義する。複数Credentialを組み合わせて提示するシナリオや、プライバシーを重視した情報開示制御に強みがあり、高度な要件定義が可能である。

両者は異なる目的とスコープを有するが、相互補完的な関係にある。特定のユースケースにおいては、両仕様の概念を組み合わせた利用も理論的に可能である。例えば、PEの`Presentation Definition`内の`input_descriptor`の`constraints`としてDCQLクエリを用いることで、PEの柔軟なフレームワーク内でDCQLの詳細なCredential属性フィルタリング能力を活用できる。

これらのクエリ言語の理解と適切な活用は、VCの相互運用性と実用性を向上させ、よりセキュアでプライバシーに配慮したデジタルアイデンティティの実現に貢献する。
