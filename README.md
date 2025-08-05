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

```bash

docker-compose up --remove-orphans

docker compose exec web rails generate devise:install

docker compose exec web rails generate devise User

docker compose exec web rails generate scaffold Folder name:string user:references is_public:boolean

docker compose exec web rails generate scaffold UserFile name:string folder:references

docker compose exec web rails active_storage:install

docker compose exec web rails generate simple_form:install --bootstrap

docker compose exec web gem install image_processing -v 1.12.2

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
```ruby
class User < ApplicationRecord
  has_many :folders, dependent: :destroy
  has_many :user_files, through: :folders
  
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable

  def accessible_files
    UserFile.joins(:folder)
            .where("folders.user_id = ? OR folders.is_public = ?", id, true)
  end
end
```
app/models/folder.rb:
```ruby
class Folder < ApplicationRecord
  belongs_to :user
  has_many :user_files, dependent: :destroy
  
  scope :public_folders, -> { where(is_public: true) }
  scope :user_folders, ->(user) { where(user: user) }
end
```
app/models/user_file.rb:
```ruby
class UserFile < ApplicationRecord
  belongs_to :folder
  has_one_attached :file

  before_validation :set_name_from_file, if: -> { file.attached? }

  # Проверяем, что файл прикреплен
  validate :file_attached

  # Check if file is previewable
  def previewable?
    return false unless file.attached?
    file.image? || (file.content_type == 'application/pdf' && file.previewable?)
  end

  # Generate preview for the file
  def file_preview
    return unless file.attached?

    begin
      if file.image?
        file.variant(resize_to_limit: [200, 200]).processed
      elsif file.content_type == 'application/pdf' && file.previewable?
        file.preview(resize_to_limit: [200, 200], format: :jpg).processed
      end
    rescue ActiveStorage::FileNotFoundError, ActiveStorage::UnpreviewableError => e
      Rails.logger.error "Preview error: #{e.message}"
      nil
    end
  end

  def high_quality_preview
    return unless file.attached?

    begin
      if file.image?
        # Для изображений
        file.variant(
          resize_to_limit: [800, 800],
          saver: { quality: 90, strip: true }
        ).processed
      end
    rescue ActiveStorage::UnpreviewableError, ActiveStorage::FileNotFoundError => e
      Rails.logger.error "Preview generation failed: #{e.message}"
      nil
    end
  end

  def icon_for_file
    return 'bi bi-file-earmark' unless file.attached?

    case file.content_type
    when /image/ then 'bi bi-image'
    when 'application/pdf' then 'bi bi-file-earmark-pdf'
    else 'bi bi-file-earmark'
    end
  end

  private

    def set_name_from_file
      self.name = file.filename.to_s
    end

    def file_attached
      errors.add(:file, "must be attached") unless file.attached?
    end
end
```
Контроллер каталогов
```ruby
class FoldersController < ApplicationController
  before_action :authenticate_user!
  before_action :set_folder, only: %i[ show edit update destroy ]
  before_action :authorize_user, only: %i[ edit update destroy ]

  def index
    @folders = Folder.all
  end

  # GET /folders/new
  def new
    @folder = Folder.new
  end

  # GET /folders/1/edit
  def edit
  end

  def create
    @folder = Folder.new(folder_params)

    respond_to do |format|
      if @folder.save
        format.html { redirect_to @folders, notice: "каталог сохранен" }
      else
        format.html { render :new, status: :unprocessable_entity }
      end
    end
  end

  def update
    respond_to do |format|
      if @folder.update(folder_params)
        format.html { redirect_to @folders, notice: "каталог сохранен" }
      else
        format.html { render :edit, status: :unprocessable_entity }
      end
    end
  end

  def destroy
    @folder.destroy!

    respond_to do |format|
      format.html { redirect_to folders_path, status: :see_other, notice: "каталог удален" }
    end
  end

  private
    # Use callbacks to share common setup or constraints between actions.
    def set_folder
      @folder = Folder.find(params[:id])
    end

    # Only allow a list of trusted parameters through.
    def folder_params
      params.require(:folder).permit(:name, :user_id, :is_public)
    end

    def authorize_user
      unless @folder.user == current_user
        redirect_to folders_path, alert: "You are not authorized to perform this action."
      end
    end
end
```
Контроллер пользовательских файлов
```ruby
class UserFilesController < ApplicationController
  before_action :authenticate_user!
  before_action :set_user_file, only: %i[ show edit update destroy ]
  before_action :authorize_user, only: %i[ edit update destroy ]

  def index
    @user_files = if params[:mine] == 'true'
      current_user.user_files
    else
      UserFile.joins(:folder)
        .where(folders: { is_public: true })
      end
    .includes(:folder, file_attachment: :blob)
  end

  def show
  end

  def new
    @user_file = UserFile.new
    @folders = current_user.folders
  end

  def edit
    @folders = current_user.folders
  end

  def create
    @user_file = current_user.user_files.build(user_file_params)
    @folders = current_user.folders

    respond_to do |format|
      if @user_file.save
        format.html { redirect_to user_file_url(@user_file), notice: "файл сохранен" }
      else
        format.html { render :new, status: :unprocessable_entity }
      end
    end
  end

  def update
    respond_to do |format|
      if @user_file.update(user_file_params)
        format.html { redirect_to user_file_url(@user_file), notice: "файл сохранен" }
      else
        format.html { render :edit, status: :unprocessable_entity }
      end
    end
  end

  def destroy
    @user_file.destroy!
    respond_to do |format|
      format.html { redirect_to user_files_url, notice: "файл удален" }
    end
  end

  private
    # Use callbacks to share common setup or constraints between actions.
    def set_user_file
      @user_file = UserFile.find(params[:id])
    end

    # Only allow a list of trusted parameters through.
    def user_file_params
      params.require(:user_file).permit(:folder_id, :file)
    end

    def authorize_user
      unless @user_file.folder.user == current_user
        redirect_to user_files_path, alert: "You are not authorized to perform this action."
      end
    end
end
```
