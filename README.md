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
Настройка форм

app/views/user_files/_form.html.erb:
```erb
<%= simple_form_for(@user_file) do |f| %>
  <%= f.error_notification %>
  <%= f.error_notification message: f.object.errors[:base].to_sentence if f.object.errors[:base].present? %>

  <div class="form-inputs mb-3">

    <%= f.association :folder, folder: current_user.folders %>
    
    <div class="mb-3">
      <%= f.label :file, class: 'form-label' %>
      <%= f.file_field :file, class: 'form-control' %>
    </div>

    <% if @user_file.file.attached? %>
      <div class="mt-3 border p-3 rounded">
        <h5>Current file:</h5>
        <% if @user_file.previewable? && @user_file.file_preview %>
          <%= image_tag @user_file.file_preview, class: 'img-thumbnail mb-2' %>
        <% end %>
        <p>
          <i class="<%= @user_file.icon_for_file %> me-2"></i>
          <%= @user_file.file.filename %>
        </p>
      </div>
    <% end %>
  </div>

  <div class="form-actions">
    <%= f.button :submit, class: 'btn btn-primary' %>
  </div>
<% end %>
```
app/views/user_files/show.html.erb
```erb
<div class="card">
  <div class="card-body">
    <h5 class="card-title"><%= @user_file.name %></h5>
    
    <% if @user_file.file.attached? %>
      <% preview = @user_file.high_quality_preview %>
      
      <% if preview %>
        <div class="mb-3">
          <%= image_tag preview, class: 'img-fluid rounded' %>
        </div>
      <% else %>
        <div class="alert alert-info">
          Preview not available for this file type
        </div>
      <% end %>

      <div class="d-flex align-items-center mb-3">
        <i class="<%= @user_file.icon_for_file %> me-2"></i>
        <span><%= @user_file.file.filename %></span>
      </div>
    <% else %>
      <div class="alert alert-warning">
        No file attached
      </div>
    <% end %>

    <%= link_to 'Download', rails_blob_path(@user_file.file, disposition: 'attachment'), 
                class: 'btn btn-primary me-2' if @user_file.file.attached? %>
    <%= link_to 'Edit', edit_user_file_path(@user_file), class: 'btn btn-secondary me-2' %>
    <%= link_to 'Back', user_files_path, class: 'btn btn-outline-secondary' %>
  </div>
</div>
```
app/views/user_files/index.html.erb
```erb
<div class="row">
  <% @user_files.each do |user_file| %>
    <div class="col-md-4 mb-4">
      <div class="card h-100">
        <% if user_file.file.attached? %>
          <% preview = user_file.high_quality_preview %>
          <% if preview %>
            <%= image_tag preview, class: 'card-img-top' %>
          <% else %>
            <div class="card-img-top bg-light d-flex align-items-center justify-content-center" style="height: 200px;">
              <i class="<%= user_file.icon_for_file %>" style="font-size: 3rem;"></i>
            </div>
          <% end %>
        <% end %>
        
        <div class="card-body">
          <h5 class="card-title">
            <%= user_file.file.filename %>
          </h5>
          <div class="card-footer bg-white">
            <%= link_to 'Show', user_file, class: 'btn btn-sm btn-outline-primary' %>
            <%= link_to 'Edit', edit_user_file_path(user_file), class: 'btn btn-sm btn-outline-secondary' %>
            <%= link_to 'Delete', user_file, 
                        method: :delete, data: { turbo_method: 'delete', turbo_confirm: "вы уверены?" }, 
                        class: 'btn btn-sm btn-outline-danger' %>
          </div>
        </div>
      </div>
    </div>
  <% end %>
</div>

<%= link_to 'New User File', new_user_file_path, class: 'btn btn-success mt-3' %>
```
app/views/layouts/application.html.erb
```erb
<!DOCTYPE html>
<html>
  <head>
    <title>Storage</title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.8.0/font/bootstrap-icons.css" rel="stylesheet">
    <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
    <%= javascript_importmap_tags %>
  </head>

  <body>
    <nav class="navbar navbar-expand-lg bg-light">
      <div class="container-fluid">
      <%= link_to "Storage", user_files_path, class: "navbar-brand" %>
        <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarSupportedContent" aria-controls="navbarSupportedContent" aria-expanded="false" aria-label="Toggle navigation">
          <span class="navbar-toggler-icon"></span>
        </button>
        <div class="collapse navbar-collapse" id="navbarSupportedContent">
          <ul class="navbar-nav me-auto mb-2 mb-lg-0">
            <form class="d-flex">
            </form>
                <li class="nav-item">
                  <%= link_to "Мои коллекции", collections_path, class: "nav-link  ? 'active' : ''}" %>
                </li>
                <li class="nav-item">
                  <%= link_to "Мои файлы", user_files_path(mine: true), class: "nav-link  ? 'active' : ''}" %>
                </li>
          </ul>
          <ul class="navbar-nav ms-auto">
            <% if user_signed_in? %>
              <li class="nav-item">
                <span class="nav-link">Вы зашли как: <strong><%= current_user.email %></strong></span>
              </li>
              <li class="nav-item">
                <%= button_to "Выйти", destroy_user_session_path, method: :delete, data: { turbo_frame: "_top" }, class: "nav-link" %>
              </li>
            <% else %>
              <li class="nav-item">
                <%= link_to "Войти", new_user_session_path, data: { turbo_frame: "_top" }, class: "nav-link" %>
              </li>
              <li class="nav-item">
                <%= link_to "Регистрация", new_user_registration_path, data: { turbo_frame: "_top" }, class: "nav-link" %>
              </li>
            <% end %>
          </ul>
        </div>
      </div>
    </nav>
    <div class="container mt-4">
      <%= yield %>
    </div>
  </body>
</html>
```
Настройка маршрутов
config/routes.rb
```ruby
Rails.application.routes.draw do
  devise_for :users
  resources :folders
  resources :user_files
  
  root to: "user_files#index"
end
```
