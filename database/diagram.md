## Схема БД

```
Table users {
    id uuid [pk]
    full_name varchar(255)
    avatar_url text
    created_at timestamptz [default: `now()`]
    updated_at timestamptz [default: `now()`]
}

Table tags {
    id uuid [pk]
    name varchar(50) [unique]
    posts_count int [default: 0]
    created_at timestamptz [default: `now()`]
    updated_at timestamptz [default: `now()`]
}

Table posts {
    id uuid [pk]
    user_id uuid [not null, ref: > users.id]
    description varchar(700)
    tag_id uuid [not null, ref: > tags.id]
    created_at timestamptz [not null, default: `now()`]
    ratings_count int [not null, default: 0]
    ratings_avg numeric(3,2)
}

Table media_asset {
    id uuid [pk]
    owner_user_id uuid [ref: > users.id]
    filename varchar(255)
    content_type varchar(100)
    file_url text [not null]
    created_at timestamptz [default: `now()`]
}

Table post_photo {
    id uuid [pk]
    post_id uuid [not null, ref: > posts.id]
    media_id uuid [not null, ref: > media_asset.id]
}

Table comments {
    id uuid [pk]
    post_id uuid [not null, ref: > posts.id]
    user_id uuid [not null, ref: > users.id]
    text varchar(500)
    created_at timestamptz [not null, default: `now()`]
}

Table ratings {
    post_id uuid [not null, ref: > posts.id]
    user_id uuid [not null, ref: > users.id]
    value int [not null]
    created_at timestamptz [not null, default: `now()`]
    updated_at timestamptz [not null, default: `now()`]
    indexes {
        (post_id,user_id) // unique
    }
}

Table follows {
    follower_user_id uuid [not null, ref: > users.id]
    target_user_id uuid [not null, ref: > users.id]
    created_at timestamptz [not null, default: `now()`]
    indexes {
        (target_user_id),
        (follower_user_id)
    }
}
```
