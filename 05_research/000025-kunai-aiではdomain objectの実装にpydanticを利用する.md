---
type: note
tags:
- '#ai/ref'
---
> [!info] 必要なセクションを選択して使用してください！

# 📜 文脈・背景

今までは複雑なロジックを含まないdata classとしてdomainsのクラスを実装していた。
しかし、ロジックが複雑になってきたため、domain objectとして実装する必要が出てきた。
domain objectは他のライブラリに依存せず純粋なpythonのコードで実装することが好まれるが、
pydancticを使えばvalidationやserialization/deserializationを簡潔に実装することができるため、実装方針を検討することとした。

## 対応範囲

1. value object
2. entity
3. aggregation

# 🎨 対応案

## 対応策1. dataclassの利用

- メリット：純粋なpythonのコードで実装できるため、ドメイン層が他のライブラリに依存しない
- デメリット：validationやserialization/deserializationを行うためのロジックを自分で実装する必要がある

## 対応策2. pydanticの利用

- メリット：validationやserialization/deserializationを行うためのロジックを自分で実装する必要がない
- デメリット
  - pydanticのバージョンアップによってドメイン層に破壊的変更が発生する可能性がある
  - 実際にv1のコードからv2では大幅な変更があった

# 🚀 決定

- 「対応策2. pydanticの利用」を採用する。
- 自前でのロジック実装を極力避け、実装速度を上げることを優先。
- ただし、pydanticの変更によるドメイン層への影響を最小化するため、domain objectの外部から直接pydanticのメソッドを呼び出さないような設計とする。
  - 下記のような基底クラスを作成し、その基底クラスを継承することでpydanticのメソッドを直接呼び出さないようにする。

```python
class ValueObject(BaseModel):
    """VOの基底クラス"""

    model_config = ConfigDict(
        frozen=True,
        extra="forbid",
        populate_by_name=True,
    )

    @classmethod
    @abstractmethod
    def create(cls: type[T], *args: tuple, **kwargs: dict) -> T:
        raise NotImplementedError

    def to_dict(self) -> dict:
        return self.model_dump(mode="json")

    @classmethod
    def from_dict(cls, data: dict) -> Self:
        return cls.model_validate(data)

    def __eq__(self, other: Any) -> bool:  # noqa: ANN401
        """
        VOの等価性判定

        - クラスが同じであること
        - 全てのフィールドが等しいこと
        """
        if not isinstance(other, self.__class__):
            return False

        # pydanticのデフォルトの等価性判定を使用
        return super().__eq__(other)
```

# 🪞 結果・影響

- value object, entity, aggregationで基底クラスを実装。
- 基底クラスを既存のdata classに適用。
- 下記のPRで対応済み
  - https://github.com/KENCOPA/kunai-ai/pull/43

# 🍜 今後の検討事項

- ドメイン層に外部ライブラリを使用するデメリットが顕在化してきた場合は、dataclassの利用を検討する。