# GraphQL::Batch

GraphQL に用意されているデータを一括取得するためのデータローダー(GraphQL::Batch)を使用し、データアクセスにおける N+1 問題を回避することで。パフォーマンスを向上させ、データへ迅速なアクセスを可能にする。

## N+1 問題とは

データベースのクエリにおいて、関連するデータを取得する際に無駄なクエリが発生する問題。一連のデータを取得するために、基本的なクエリ(1 回目のクエリ)を実行し、その後、関連するデータを取得するために各データごとに個別のクエリ(n 回のクエリ)が発生する。

**例：）ユーザーの一覧を取得する際に、各ユーザーの投稿を取得する必要がある場合**

最初にユーザー取得するためのクエリ(1 回目のクエリ)が発行される。そして、各ユーザーの投稿を取得するために、ユーザーの数だけ追加のクエリ(n 回目のクエリ)が発行されることになる。

この問題の結果として、データベースに対して冗長なクエリが発生し、パフォーマンスの低下や遅延が生じる可能性がある。

## 実装例（[food-app](https://github.com/DaisukeKarasawa/food-app)）

### GraphQL::Batch のインストール

```
# Gemfile

gem "graphql-batch" # 追加
```

### GraphQL::Batch の使用宣言

スキーマで GraphQL::Batch を使用することを宣言する。

```
# app/graphql/administration_food_app_schema.rb

class AdministrationFoodAppSchema < GraphQL::Schema
    mutation(Types::MutationType)
    query(Types::QueryType)
    use GraphQL::Batch   # 追加

    # ...省略
end
```

### サンプルコードを使用

アソシエーション用の[AssociationLoader クラスのサンプルコード](https://github.com/Shopify/graphql-batch/tree/master)を 'app/graphql/loaders/association_loader.rb' に記述する。

```
# app/graphql/loaders/association_loader.rb

module Loaders
    class AssociationLoader < GraphQL::Batch::Loader
        def self.validate(model, association_name)
            new(model, association_name)
            nil
        end

        def initialize(model, association_name)
            super()
            @model = model
            @association_name = association_name
            validate
        end

        def load(record)
            raise TypeError, "#{@model} loader can't load association for #{record.class}" unless record.is_a?(@model)
            return Promise.resolve(read_association(record)) if association_loaded?(record)
            super
        end

        # We want to load the associations on all records, even if they have the same id
        def cache_key(record)
            record.object_id
        end

        def perform(records)
            preload_association(records)
            records.each { |record| fulfill(record, read_association(record)) }
        end

        private

        def validate
            unless @model.reflect_on_association(@association_name)
                raise ArgumentError, "No association #{@association_name} on #{@model}"
            end
        end

        def preload_association(records)
            ::ActiveRecord::Associations::Preloader.new(records: records, associations: @association_name).call
        end

        def read_association(record)
            record.public_send(@association_name)
        end

        def association_loaded?(record)
            record.association(@association_name).loaded?
        end
    end
end
```

### AssociationLoader を使用する

AssociationLoader を使用するように変更する。

```
# app/graphql/types/dish_type.rb

module Types
    class DishType < Types::BaseObject
        field :id, ID, null: false
        field :name, String, null: false
        field :foods, [Types::FoodType], null: true
        field :recipe_urls, [Types::RecipeUrlType], null: true

        # 以下のコードを追加
        def recipe_urls
            Loaders::AssociationLoader.for(Dish, :recipe_urls).load(object)
        end
    end
end
```

## 結果例(料理の一覧取得クエリ)

**- 実装前 -**

料理のデータには'カレー'と'野菜炒め'があり、それぞれに紐づいている URL へアクセスするための同じ 'SELECT 文'が 2 度も実行されている。
故に、料理のデータ量が増えれば増えるほど、URL 取得のクエリが増えていくので、結果、パフォーマンスの低下や遅延が生じる可能性がある。

```
// ログ

SELECT "schema_migrations"."version" FROM "schema_migrations" ORDER BY "schema_migrations"."version" ASC...
SELECT "dishes".* FROM "dishes"
SELECT "recipe_urls".* FROM "recipe_urls" WHERE "recipe_urls"."dish_id" = ?  [["dish_id", 1]]
SELECT "recipe_urls".* FROM "recipe_urls" WHERE "recipe_urls"."dish_id" = ?  [["dish_id", 2]]
```

**- 実装後 -**

URL 取得クエリを一度の実行で、それぞれの料理の全ての URL を取得できるようになった。

```
// ログ

SELECT "schema_migrations"."version" FROM "schema_migrations" ORDER BY "schema_migrations"."version" ASC ...
SELECT "dishes".* FROM "dishes"
SELECT "recipe_urls".* FROM "recipe_urls" WHERE "recipe_urls"."dish_id" IN (?, ?)
```
