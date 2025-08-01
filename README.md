```bash
mkdir storage

cd storage

wget https://raw.githubusercontent.com/YouraSilin/FileStorage/refs/heads/master/Dockerfile

wget https://raw.githubusercontent.com/YouraSilin/FileStorage/refs/heads/master/Gemfile

wget https://raw.githubusercontent.com/YouraSilin/FileStorage/refs/heads/master/Procfile.dev

wget https://raw.githubusercontent.com/YouraSilin/FileStorage/refs/heads/master/docker-compose.yml

wget https://raw.githubusercontent.com/YouraSilin/FileStorage/refs/heads/master/entrypoint.sh

wget https://raw.githubusercontent.com/YouraSilin/FileStorage/refs/heads/master/package.json

touch Gemfile.lock

docker compose build

docker compose run --no-deps web rails new . --force --database=postgresql --css=bootstrap

sudo chown -R $USER:$USER .
```
replace this files

https://github.com/YouraSilin/FileStorage/blob/master/config/database.yml

https://github.com/YouraSilin/FileStorage/blob/master/Dockerfile

https://github.com/YouraSilin/FileStorage/blob/master/Gemfile

docker compose up

docker compose exec web rails generate devise:install

docker compose exec web rails generate devise User

docker compose exec web rails generate scaffold Collection name:string user:references is_public:boolean

docker compose exec web rails generate scaffold UserFile name:string collection:references

docker compose exec web rails active_storage:install

docker compose exec web rails generate simple_form:install --bootstrap

docker compose exec web rake db:create db:migrate

sudo chown -R $USER:$USER .
```
Here is a possible configuration for config/environments/development.rb:
```erb
config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }
```
Прописать mini-magic в /config/environments/development.rb
```ruby
  config.active_storage.variant_processor = :mini_magick
```
Модификация моделей

app/models/user.rb:
class User < ApplicationRecord
  has_many :collections, dependent: :destroy
  has_many :user_files, through: :collections
  
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable

  def accessible_files
    UserFile.joins(:collection)
            .where("collections.user_id = ? OR collections.is_public = ?", id, true)
  end
end
```
app/models/collection.rb:
```ruby
class Collection < ApplicationRecord
  belongs_to :user
  has_many :user_files, dependent: :destroy
  
  scope :public_collections, -> { where(is_public: true) }
  scope :user_collections, ->(user) { where(user: user) }
end
```
app/models/user_file.rb:
```ruby
class UserFile < ApplicationRecord
  belongs_to :collection
  has_one_attached :file

  validates :name, presence: true

  # Check if file is previewable (images or PDFs)
  def previewable?
    return false unless file.attached?
    file.image? || file.content_type == 'application/pdf'
  end

  # Generate preview for the file
  def file_preview
    return unless file.attached?

    if file.image?
      file.variant(resize_to_limit: [200, 200]).processed
    elsif file.content_type == 'application/pdf'
      file.preview(resize_to_limit: [200, 200]).processed
    end
  rescue ActiveStorage::FileNotFoundError
    nil
  end

  # Get appropriate icon for the file type
  def icon_for_file
    if file.image?
      'bi bi-image'
    elsif file.content_type == 'application/pdf'
      'bi bi-file-earmark-pdf'
    else
      'bi bi-file-earmark'
    end
  end
end
```
