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

docker-compose build --no-cache

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

    if file.image?
      file.variant(resize_to_limit: [200, 200]).processed
    
    elsif file.content_type == 'application/pdf' && file.previewable?
      file.preview(resize_to_limit: [200, 200], format: :jpg).processed
    end
    rescue ActiveStorage::FileNotFoundError, ActiveStorage::UnpreviewableError => e
        Rails.logger.error "Preview error: #{e.message}"
        nil
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
  before_action :authorize_owner, only: %i[ edit update destroy ]

  # GET /folders or /folders.json
  def index
    @folders = if params[:mine] == 'true'
                 current_user.folders
               else
                 Folder.public_folders
               end
  end

  # GET /folders/1 or /folders/1.json
  def show
    @folder = Folder.find(params[:id])
  end

  # GET /folders/new
  def new
    @folder = Folder.new
  end

  # GET /folders/1/edit
  def edit
  end

  # POST /folders or /folders.json
  def create
    @folder = Folder.new(folder_params)

    respond_to do |format|
      if @folder.save
        format.html { redirect_to @folder, notice: "Folder was successfully created." }
        format.json { render :show, status: :created, location: @folder }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: @folder.errors, status: :unprocessable_entity }
      end
    end
  end

  # PATCH/PUT /folders/1 or /folders/1.json
  def update
    respond_to do |format|
      if @folder.update(folder_params)
        format.html { redirect_to @folder, notice: "Folder was successfully updated." }
        format.json { render :show, status: :ok, location: @folder }
      else
        format.html { render :edit, status: :unprocessable_entity }
        format.json { render json: @folder.errors, status: :unprocessable_entity }
      end
    end
  end

  # DELETE /folders/1 or /folders/1.json
  def destroy
    @folder.destroy!

    respond_to do |format|
      format.html { redirect_to folders_path, status: :see_other, notice: "Folder was successfully destroyed." }
      format.json { head :no_content }
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

    def authorize_owner
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
    @user_files =
      if params[:mine] == 'true'
        current_user.user_files.includes(:folder, file_attachment: :blob)
      else
        UserFile.joins(:folder).merge(Folder.public_folders).includes(:folder, file_attachment: :blob)
      end
  end

  def show
  end

  def new
    @user_file = UserFile.new
    @user_file.folder_id = params[:folder_id] if params[:folder_id]
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
        format.html { redirect_to user_files_path, notice: "файл сохранен" }
      else
        format.html { render :new, status: :unprocessable_entity }
      end
    end
  end

  def update
    respond_to do |format|
      if @user_file.update(user_file_params)
        format.html { redirect_to user_files_path, notice: "файл сохранен" }
      else
        format.html { render :edit, status: :unprocessable_entity }
      end
    end
  end

  def destroy
    @user_file.destroy!
    respond_to do |format|
      format.html { redirect_to user_files_path, notice: "файл удален" }
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
    if @user_file.folder.nil? || @user_file.folder.user != current_user
      redirect_to user_files_path, 
                 alert: "You are not authorized to perform this action.",
                 status: :forbidden
    end
  end
end
```
Настройка форм

app/views/user_files/_form.html.erb:
```erb
<%= simple_form_for(@user_file) do |f| %>
  <%= f.error_notification %>

  <div class="form-inputs">
    <% if @user_file.folder_id.present? %>
      <%= f.input :folder_id, as: :hidden %>
      <div class="alert alert-info mb-3">
        File will be added to:
        <strong><%= link_to user_file.folder.name, user_file.folder, class: 'link-dark' %></strong>
      </div>
    <% else %>
      <%= f.association :folder, folder: @folders %>
    <% end %>

    <%= f.input :file, as: :file%>
  </div>

  <div class="form-actions mt-3">
    <%= f.button :submit, 'Upload File', class: 'btn btn-primary' %>
    <%= link_to 'Cancel', @user_file.folder || folders_path, class: 'btn btn-outline-secondary' %>
  </div>
<% end %>
```
app/views/user_files/show.html.erb
```erb
<div class="card">
  <div class="card-body">
    <h5 class="card-title">
      <div class="d-flex align-items-center mb-3">
        <i class="<%= @user_file.icon_for_file %> fs-3 me-2"></i>
        <span><%= @user_file.file.filename %></span>
      </div>
    </h5>
    
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

      
    <% else %>
      <div class="alert alert-warning">
        No file attached
      </div>
    <% end %>
         
      <div class="d-flex align-items-center mb-3">

      <i class="bi bi-folder<%= @user_file.folder.is_public ? '' : '-x' %> fs-3 me-3"></i>
          <div>
            <h5><%= link_to @user_file.folder.name, @user_file.folder, class: 'link-dark text-decoration-none' %></h5>
            <span class="badge bg-<%= @user_file.folder.is_public ? 'success' : 'secondary' %>">
              <%= @user_file.folder.is_public ? 'Public' : 'Private' %>
            </span>
          </div>
      </div>
      <div class="text-muted small mt-2">
        Owner: <%= @user_file.folder.user.email %>
      </div>
    
    <div class="card-footer bg-white">        
      <%= link_to 'Download', rails_blob_path(@user_file.file, disposition: 'attachment'), 
          class: 'btn btn-primary me-2' if @user_file.file.attached? %>
      <% if @user_file.folder.user == current_user %>
        <%= link_to 'Edit', edit_user_file_path(@user_file), class: 'btn btn-outline-secondary' %>
        <%= link_to 'Delete', @user_file, 
                    method: :delete, data: { turbo_method: 'delete', turbo_confirm: "вы уверены?" }, 
                    class: 'btn btn-outline-danger' %>
      <% end %>
      <%= link_to 'Back', user_files_path, class: 'btn btn-outline-primary' %>
    </div>
  </div>
</div>
```
app/views/user_files/index.html.erb
```erb
<div class="row row-cols-1 row-cols-sm-2 row-cols-md-3 row-cols-lg-4 g-4">
  <% @user_files.each do |user_file| %>
    <div class="col">
      <div class="card h-100">
        <% if user_file.file.attached? %>
          <% preview = user_file.high_quality_preview %>
          <% if preview %>
            <div class="card-img-top-container">
              <%= image_tag preview, class: 'card-img-top' %>
            </div>
          <% else %>
            <div class="card-img-top bg-light d-flex align-items-center justify-content-center" style="height: 150px;">
              <i class="<%= user_file.icon_for_file %> fs-1"></i>
            </div>
          <% end %>
        <% end %>
        
        <div class="card-body d-flex flex-column">
          <h5 class="card-title text-truncate" title="<%= user_file.file.filename %>">
            <%= user_file.file.filename %>
          </h5>
          
          <div class="d-flex align-items-center mb-2">
            <i class="bi bi-folder<%= user_file.folder.is_public ? '' : '-x' %> me-2"></i>
            <small class="text-muted"><%= link_to user_file.folder.name, user_file.folder, class: 'link-dark text-decoration-none' %></small>      
          </div>

          <div class="mt-auto">
            <div class="d-flex justify-content-between">
              <%= link_to 'Show', user_file, class: 'btn btn-outline-primary' %>
              <% if user_file.folder.user == current_user %>
                <%= link_to 'Edit', edit_user_file_path(user_file), class: 'btn btn-outline-secondary' %>
                <%= link_to 'Delete', user_file, 
                            method: :delete, 
                            data: { turbo_method: 'delete', turbo_confirm: "Are you sure?" }, 
                            class: 'btn btn-outline-danger' %>
              <% end %>
            </div>            
          </div>
        </div>
      </div>
    </div>
  <% end %>
</div>
```
app/views/folders/_folder.html.erb
```erb
<div class="col-md-3 mb-4 folder-card" id="<%= dom_id folder %>">
  <div class="card h-100">
    <%= link_to folder, class: 'link-dark text-decoration-none stretched-link', style: 'z-index: 1;' do %>
      <div class="card-body text-center">
        <div class="folder-icon mb-3">
          <i class="bi bi-folder<%= folder.is_public ? '' : '-x' %> fs-1"></i>
        </div>
        <h5 class="card-title text-dark"><%= folder.name %></h5>
        <div class="badge bg-<%= folder.is_public ? 'success' : 'secondary' %>">
          <%= folder.is_public ? 'Public' : 'Private' %>
        </div>
      </div>
    <% end %>
    <% if folder.user == current_user %>
      <div class="card-footer bg-white position-relative" style="z-index: 2;">
        <%= link_to 'Edit', edit_folder_path(folder), class: 'btn btn-outline-secondary' %>
        <%= link_to 'Delete', folder, 
                    method: :delete, 
                    data: { turbo_method: 'delete', turbo_confirm: "вы уверены?" }, 
                    class: 'btn btn-outline-danger' %>
      </div>
    <% end %>
  </div>
</div>
```
app/views/folders/_form.html.erb
```erb
<%= simple_form_for(folder) do |f| %>
  <%= f.error_notification %>
  <%= f.error_notification message: f.object.errors[:base].to_sentence if f.object.errors[:base].present? %>

  <div class="form-inputs mb-4">
    <%= f.input :name, label: 'Folder Name', 
                input_html: { class: 'form-control', placeholder: 'Enter folder name' } %>
    
    <%= f.input :is_public, label: 'Make this folder public?',
                wrapper_class: 'form-check',
                input_html: { class: 'form-check-input' },
                label_html: { class: 'form-check-label' } %>
    
    <%= f.hidden_field :user_id, value: current_user.id %>
  </div>

  <div class="form-actions">
    <%= f.button :submit, class: 'btn btn-primary' %>
    <%= link_to "Cancel", folders_path, class: 'btn btn-outline-secondary ms-2' %>
  </div>
<% end %>
```
app/views/folders/index.html.erb
```erb
<div class="container mt-4">
  <div class="mb-4">
    <%= link_to new_folder_path, class: 'btn btn-primary' do %>
      New Folder
    <% end %>
  </div>
  <% if notice.present? %>
    <div class="alert alert-success alert-dismissible fade show">
      <%= notice %>
      <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
    </div>
  <% end %>

  <div class="row">
    <%= render @folders || 'No folders found' %>
  </div>
</div>
```
app/views/folders/show.html.erb
```erb
<div class="container mt-4">
  <% if notice.present? %>
    <div class="alert alert-success alert-dismissible fade show">
      <%= notice %>
      <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
    </div>
  <% end %>

  <div class="card mb-4">
    <div class="card-body">
      <div class="d-flex align-items-center mb-3">
        <i class="bi bi-folder<%= @folder.is_public ? '' : '-x' %> fs-1 me-3"></i>
        <div>
          <h2 class="card-title mb-0"><%= @folder.name %></h2>
          <span class="badge bg-<%= @folder.is_public ? 'success' : 'secondary' %>">
            <%= @folder.is_public ? 'Public' : 'Private' %>
          </span>
        </div>
      </div>
      
      <div class="d-flex flex-wrap gap-2 mt-3">
        <% if @folder.user == current_user %>
          <%= link_to 'Add File', new_user_file_path(folder_id: @folder.id), class: 'btn btn-success' %>
          <%= link_to 'Edit', edit_folder_path(@folder), class: 'btn btn-outline-secondary' %>
          <%= link_to 'Delete', @folder, 
                      method: :delete, 
                      data: { turbo_method: 'delete', turbo_confirm: "Are you sure?" }, 
                      class: 'btn btn-outline-danger' %>
        <% end %>
        <%= link_to 'Back', folders_path, class: 'btn btn-outline-primary' %>
      </div>
    </div>
  </div>

  <h3 class="mb-4">
    <i class="bi bi-files me-2"></i>
    Files in this folder
  </h3>
  
  <% if @folder.user_files.any? %>
    <div class="row row-cols-1 row-cols-sm-2 row-cols-md-3 row-cols-lg-4 g-4">
      <% @folder.user_files.each do |user_file| %>
        <div class="col">
          <div class="card h-100">
            <% if user_file.file.attached? %>
              <% preview = user_file.high_quality_preview %>
              <% if preview %>
                <div class="card-img-top-container">
                  <%= image_tag preview, class: 'card-img-top' %>
                </div>
              <% else %>
                <div class="card-img-top bg-light d-flex align-items-center justify-content-center" style="height: 150px;">
                  <i class="<%= user_file.icon_for_file %> fs-1"></i>
                </div>
              <% end %>
            <% end %>
            
            <div class="card-body d-flex flex-column">
              <h5 class="card-title text-truncate" title="<%= user_file.file.filename %>">
                <%= user_file.file.filename %>
              </h5>
              
              <div class="mt-auto">
                <div class="d-flex justify-content-between">
                  <%= link_to 'Show', user_file, class: 'btn btn-outline-primary' %>
                  <% if @folder.user == current_user %>
                    <%= link_to 'Edit', edit_user_file_path(user_file), class: 'btn btn-outline-secondary' %>
                    <%= link_to 'Delete', user_file, 
                                method: :delete, 
                                data: { turbo_method: 'delete', turbo_confirm: "Are you sure?" }, 
                                class: 'btn btn-outline-danger' %>
                  <% end %>
                </div>
              </div>
            </div>
          </div>
        </div>
      <% end %>
    </div>
  <% else %>
    <div class="alert alert-info">
      <i class="bi bi-info-circle me-2"></i>
      This folder is empty. Add your first file.
    </div>
  <% end %>
</div>
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
                  <%= link_to "Все папки", folders_path, class: "nav-link  ? 'active' : ''}" %>
                </li>
                <li class="nav-item">
                  <%= link_to "Мои папки", folders_path(mine: true), class: "nav-link  ? 'active' : ''}" %>
                </li>
                <li class="nav-item">
                  <%= link_to "Все файлы", user_files_path, class: "nav-link  ? 'active' : ''}" %>
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
Добавить слили в application.bootstrap.scss
```css
/* Стили для папок */
.folder-card {
  transition: transform 0.2s;
}
.folder-card:hover {
  transform: scale(1.1);
}

/* Cards layout */
.card-img-top-container {
  height: 200px;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #f8f9fa;
  
  .card-img-top {
    object-fit: contain;
    max-height: 100%;
    width: auto;
  }
}

/* Responsive adjustments */
@media (max-width: 576px) {
  .card-img-top-container {
    height: 150px;
  }
  
  .card-body {
    padding: 1rem;
  }
}

/* Ensure buttons stay in one line */
.btn-group-vertical {
  flex-direction: row !important;
  gap: 0.5rem;
}
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
Добавление множественной загрузки файлов

Создадим Stimulus контроллер:
```bash
docker compose exec web rails generate stimulus file_upload
```
Редактируем созданный контроллер app/javascript/controllers/file_upload_controller.js:
```js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["fileInput", "submit", "folderSelect"]
  
  connect() {
    console.log("File Upload Controller connected")
  }

  async submitForm(event) {
    event.preventDefault()
    this.submitTarget.disabled = true
    const files = Array.from(this.fileInputTarget.files)
    const folderId = this.folderSelectTarget.value
    
    const originalButtonText = this.submitTarget.innerHTML
    this.submitTarget.innerHTML = `
      <span class="spinner-border spinner-border-sm me-2" role="status" aria-hidden="true"></span>
      Uploading ${files.length} files...
    `
    
    try {
      for (const file of files) {
        await this.uploadSingleFile(file, folderId)
      }
      Turbo.visit(window.location.href, { action: 'replace' })
    } catch (error) {
      console.error('Upload error:', error)
      alert(`Error uploading files: ${error.message}`)
      this.submitTarget.innerHTML = originalButtonText
      this.submitTarget.disabled = false
    }
  }

  async uploadSingleFile(file, folderId) {
    const formData = new FormData()
    formData.append('user_file[file]', file)
    formData.append('user_file[folder_id]', folderId)
    
    const response = await fetch(this.element.action, {
      method: 'POST',
      body: formData,
      headers: {
        'X-CSRF-Token': document.querySelector("[name='csrf-token']").content,
        'Accept': 'application/json'
      }
    })
    
    if (!response.ok) {
      const error = await response.json().catch(() => ({ error: 'Upload failed' }))
      throw new Error(error.error || 'Upload failed')
    }
  }
}
```
В файле app/javascript/controllers/index.js должно быть:
```js
// Import and register all your controllers from the importmap via controllers/**/*_controller
import { application } from "controllers/application"
import { eagerLoadControllersFrom } from "@hotwired/stimulus-loading"
eagerLoadControllersFrom("controllers", application)

import FileUploadController from "./file_upload_controller"
application.register("file-upload", FileUploadController)
```
Убедимся, что в config/importmap.rb есть:
```ruby
# Pin npm packages by running ./bin/importmap

pin "application"
pin "@hotwired/turbo-rails", to: "turbo.min.js"
pin "@hotwired/stimulus", to: "stimulus.min.js"
pin "@hotwired/stimulus-loading", to: "stimulus-loading.js"
pin_all_from "app/javascript/controllers", under: "controllers"
pin "bootstrap", to: "bootstrap.bundle.min.js"

pin_all_from "app/javascript/controllers", under: "controllers"
```
Обновляем форму app/views/user_files/_form.html.erb:
```erb
<%= simple_form_for(@user_file, 
    html: { 
      data: { 
        controller: "file-upload",
        action: "submit->file-upload#submitForm"
      }
    }) do |f| %>
  
  <%= f.error_notification %>

  <div class="form-inputs">
    <% if @user_file.folder_id.present? %>
      <%= f.hidden_field :folder_id, data: { file_upload_target: "folderSelect" } %>
      <div class="alert alert-info mb-3">
        Files will be added to: <strong><%= @user_file.folder.name %></strong>
      </div>
    <% else %>
      <%= f.association :folder, 
          collection: current_user.folders,
          input_html: { 
            data: { file_upload_target: "folderSelect" },
            class: 'form-select'
          } %>
    <% end %>

    <div class="mb-3">
      <%= f.label :file, "Select files", class: "form-label" %>
      <%= f.file_field :file, 
          multiple: true,
          direct_upload: false,
          class: 'form-control',
          data: { file_upload_target: "fileInput" } %>
      <div class="form-text">Hold Ctrl/Cmd to select multiple files</div>
    </div>
  </div>

  <div class="form-actions mt-3">
    <%= f.button :button, 
                type: 'submit',
                class: 'btn btn-primary',
                data: { file_upload_target: "submit" },
                disabled: false do %>
      Upload Files
    <% end %>
    <%= link_to 'Cancel', @user_file.folder || folders_path, class: 'btn btn-outline-secondary' %>
  </div>
<% end %>
```
Добавим стили для загрузки в app/assets/stylesheets/application.bootstrap.scss
```css
/* Анимация загрузки */
@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.upload-spinner {
  display: inline-block;
  width: 1rem;
  height: 1rem;
  border: 2px solid rgba(0,0,0,0.1);
  border-radius: 50%;
  border-top-color: #0d6efd;
  animation: spin 1s ease-in-out infinite;
  margin-right: 0.5rem;
}

/* Upload button loading state */
.btn .spinner-border {
  vertical-align: text-top;
}
```
Обновим контроллер UserFilesController:
```erb
def create
    @user_file = current_user.user_files.build(user_file_params)
    @folders = current_user.folders

    respond_to do |format|
      if @user_file.save
        format.html { redirect_to user_files_path, notice: "File was successfully uploaded." }
        format.json { render :show, status: :created, location: @user_file }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: @user_file.errors, status: :unprocessable_entity }
      end
    end
  end
```
Добавление различных способов загрузки
Сначала добавим необходимые гемы в Gemfile:
```ruby
gem 'down', '~> 5.4.1' # Для загрузки файлов по URL
gem 'marcel' # Для определения MIME типа
```
```bash
docker compose exec web rails bundle install
docker compose exec web rails generate stimulus file_uploader
sudo chown -R $USER:$USER .
```
Редактируем app/javascript/controllers/file_uploader_controller.js:
```js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["dropzone", "fileInput", "urlInput", "urlForm", "progress", "preview", "submit"]
  
  connect() {
    this.setupDropzone()
    this.setupPasteHandler()
    console.log("File Uploader Controller connected")
  }

  // Триггер выбора файлов
  triggerFileSelect() {
    this.fileInputTarget.click()
  }

  // Обработка выбора файлов через input
  handleFileSelect(event) {
    const files = event.target.files
    if (files && files.length > 0) {
      this.handleFiles(files)
      event.target.value = '' // Сброс значения для возможности повторной загрузки тех же файлов
    }
  }
  
  // Настройка drag and drop
  setupDropzone() {
    this.dropzoneTarget.addEventListener('dragover', (e) => {
      e.preventDefault()
      this.dropzoneTarget.classList.add('dragover')
    })

    this.dropzoneTarget.addEventListener('dragleave', () => {
      this.dropzoneTarget.classList.remove('dragover')
    })

    this.dropzoneTarget.addEventListener('drop', (e) => {
      e.preventDefault()
      this.dropzoneTarget.classList.remove('dragover')
      
      if (e.dataTransfer.files.length > 0) {
        this.handleFiles(e.dataTransfer.files)
      } else if (e.dataTransfer.getData('text')) {
        this.handlePotentialUrl(e.dataTransfer.getData('text'))
      }
    })
  }

  // Обработчик вставки из буфера обмена
  setupPasteHandler() {
    document.addEventListener('paste', (e) => {
      if (e.clipboardData.files.length > 0) {
        this.handleFiles(e.clipboardData.files)
      } else if (e.clipboardData.getData('text')) {
        this.handlePotentialUrl(e.clipboardData.getData('text'))
      }
    })
  }

  // Обработка выбора файлов через input
  handleFileSelect(event) {
    if (event.target.files.length > 0) {
      this.handleFiles(event.target.files)
      event.target.value = '' // Сброс значения для возможности повторной загрузки тех же файлов
    }
  }

  // Обработка URL из формы
  handleUrlSubmit(event) {
    event.preventDefault()
    const url = this.urlInputTarget.value.trim()
    if (url) {
      this.downloadFromUrl(url)
      this.urlInputTarget.value = ''
    }
  }

  // Проверка, является ли текст URL
  handlePotentialUrl(text) {
    if (this.isValidUrl(text)) {
      if (confirm(`Download file from ${text}?`)) {
        this.downloadFromUrl(text)
      }
    }
  }

  // Валидация URL
  isValidUrl(string) {
    try {
      new URL(string)
      return true
    } catch (_) {
      return false
    }
  }

  // Загрузка файлов по URL
  async downloadFromUrl(url) {
    this.showProgress(`Downloading from ${url}...`)
    
    try {
      const response = await fetch('/file_uploader/download', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-CSRF-Token': document.querySelector("[name='csrf-token']").content
        },
        body: JSON.stringify({ 
          url: url, 
          folder_id: this.folderId 
        })
      })

      const data = await response.json()
      
      if (response.ok) {
        this.addPreview(data.file)
      } else {
        throw new Error(data.error || 'Download failed')
      }
    } catch (error) {
      console.error('Download error:', error)
      alert(`Error downloading file: ${error.message}`)
    } finally {
      this.hideProgress()
    }
  }

  // Обработка локальных файлов
  async handleFiles(files) {
    for (let i = 0; i < files.length; i++) {
      const file = files[i]
      this.showProgress(`Uploading ${file.name}...`)

      try {
        const formData = new FormData()
        formData.append('user_file[file]', file)
        if (this.folderId) {
          formData.append('user_file[folder_id]', this.folderId)
        }

        const response = await fetch(this.element.action, {
          method: 'POST',
          body: formData,
          headers: {
            'X-CSRF-Token': document.querySelector("[name='csrf-token']").content,
            'Accept': 'application/json'
          }
        })

        const data = await response.json()
        
        if (response.ok) {
          this.addPreview(data.file)
        } else {
          throw new Error(data.error || 'Upload failed')
        }
      } catch (error) {
        console.error('Upload error:', error)
        alert(`Error uploading ${file.name}: ${error.message}`)
      } finally {
        this.hideProgress()
      }
    }
  }

  // Добавление превью загруженного файла
  addPreview(fileData) {
    const previewItem = document.createElement('div')
    previewItem.className = 'preview-item'
    previewItem.innerHTML = `
      <div class="card mb-2">
        <div class="card-body">
          <div class="d-flex align-items-center">
            <i class="${this.getFileIcon(fileData.content_type)} me-2"></i>
            <span>${fileData.filename}</span>
          </div>
        </div>
      </div>
    `
    this.previewTarget.appendChild(previewItem)
  }

  // Получение иконки для типа файла
  getFileIcon(contentType) {
    if (contentType.startsWith('image/')) return 'bi bi-image'
    if (contentType === 'application/pdf') return 'bi bi-file-earmark-pdf'
    return 'bi bi-file-earmark'
  }

  // Показать прогресс
  showProgress(message) {
    this.progressTarget.textContent = message
    this.progressTarget.style.display = 'block'
    this.submitTarget.disabled = true
  }

  // Скрыть прогресс
  hideProgress() {
    this.progressTarget.style.display = 'none'
    this.submitTarget.disabled = false
  }

  get folderId() {
    return this.data.get('folderId') || document.querySelector('#user_file_folder_id')?.value
  }
}
```
В app/javascript/controllers/index.js должно быть:
```js
// Import and register all your controllers from the importmap via controllers/**/*_controller
import { application } from "controllers/application"
import { eagerLoadControllersFrom } from "@hotwired/stimulus-loading"
eagerLoadControllersFrom("controllers", application)

import FileUploadController from "./file_upload_controller"
application.register("file-upload", FileUploadController)

import FileUploaderController from "./file_uploader_controller"
application.register("file-uploader", FileUploaderController)
```
В config/importmap.rb должно быть:
```ruby
# Pin npm packages by running ./bin/importmap

pin "application"
pin "@hotwired/turbo-rails", to: "turbo.min.js"
pin "@hotwired/stimulus", to: "stimulus.min.js"
pin "@hotwired/stimulus-loading", to: "stimulus-loading.js"
pin_all_from "app/javascript/controllers", under: "controllers"
pin "bootstrap", to: "bootstrap.bundle.min.js"

pin_all_from "app/javascript/controllers", under: "controllers"
```
Добавим маршрут для обработки загрузки по URL в config/routes.rb:
```ruby
Rails.application.routes.draw do
  devise_for :users
  resources :folders
  resources :user_files
  
  post '/file_uploader/download', to: 'file_uploader#download'
  
  root to: "user_files#index"
end
```
Создадим контроллер для обработки загрузки по URL:
```bash
docker compose exec web rails generate controller FileUploader
sudo chown -R $USER:$USER .
```
Редактируем app/controllers/file_uploader_controller.rb:
```ruby
class FileUploaderController < ApplicationController
  before_action :authenticate_user!

  def download
    url = params[:url]
    folder_id = params[:folder_id]

    begin
      tempfile = Down.download(url)
      user_file = current_user.user_files.build(folder_id: folder_id)
      user_file.file.attach(io: tempfile, filename: File.basename(tempfile.path))

      if user_file.save
        render json: { 
          file: {
            filename: user_file.file.filename.to_s,
            content_type: user_file.file.content_type,
            url: url_for(user_file.file)
          }
        }, status: :created
      else
        render json: { error: user_file.errors.full_messages.join(', ') }, status: :unprocessable_entity
      end
    rescue Down::Error => e
      render json: { error: "Failed to download file: #{e.message}" }, status: :unprocessable_entity
    ensure
      tempfile.close! if tempfile
    end
  end
end
```
Обновим форму загрузки файлов app/views/user_files/_form.html.erb:
```erb
<div class="col-md-3 mb-4 folder-card" id="<%= dom_id folder %>">
  <div class="card h-100">
    <%= link_to folder, class: 'link-dark text-decoration-none stretched-link', style: 'z-index: 1;' do %>
      <div class="card-body text-center">
        <div class="folder-icon mb-3">
          <i class="bi bi-folder<%= folder.is_public ? '' : '-x' %> fs-1"></i>
        </div>
        <h5 class="card-title text-dark"><%= folder.name %></h5>
        <div class="badge bg-<%= folder.is_public ? 'success' : 'secondary' %>">
          <%= folder.is_public ? 'Public' : 'Private' %>
        </div>
      </div>
    <% end %>
    <% if folder.user == current_user %>
      <div class="card-footer bg-white position-relative" style="z-index: 2;">
        <%= link_to 'Edit', edit_folder_path(folder), class: 'btn btn-outline-secondary' %>
        <%= link_to 'Delete', folder, 
                    method: :delete, 
                    data: { turbo_method: 'delete', turbo_confirm: "вы уверены?" }, 
                    class: 'btn btn-outline-danger' %>
      </div>
    <% end %>
  </div>
</div>
```
Добавим стили в app/assets/stylesheets/application.bootstrap.scss:
```css
/* File Uploader Styles */
.file-uploader-container {
  .dropzone {
    transition: all 0.3s;
    background-color: #f8f9fa;
    cursor: pointer;
    
    &:hover {
      background-color: #e9ecef;
    }
    
    &.dragover {
      background-color: #e2e6ea;
      border-color: #0d6efd;
    }
  }
  
  .preview-container {
    max-height: 300px;
    overflow-y: auto;
    
    .preview-item {
      &:not(:last-child) {
        margin-bottom: 0.5rem;
      }
    }
  }
  
  .progress-indicator {
    &:before {
      content: '';
      display: inline-block;
      width: 1rem;
      height: 1rem;
      border: 2px solid rgba(0,0,0,0.1);
      border-radius: 50%;
      border-top-color: #0d6efd;
      animation: spin 1s linear infinite;
      margin-right: 0.5rem;
    }
  }
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
```
Обновим контроллер UserFilesController:
```ruby
class UserFilesController < ApplicationController
  before_action :authenticate_user!
  before_action :set_user_file, only: %i[ show edit update destroy ]
  before_action :authorize_user, only: %i[ edit update destroy ]

  def index
    @user_files =
      if params[:mine] == 'true'
        current_user.user_files.includes(:folder, file_attachment: :blob)
      else
        UserFile.joins(:folder).merge(Folder.public_folders).includes(:folder, file_attachment: :blob)
      end
  end

  def show
  end

  def new
    @user_file = UserFile.new
    @user_file.folder_id = params[:folder_id] if params[:folder_id]
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
        format.html { redirect_to user_files_path, notice: "File was successfully uploaded." }
        format.json { render json: { 
          file: {
            filename: @user_file.file.filename.to_s,
            content_type: @user_file.file.content_type,
            url: url_for(@user_file.file)
          }
        }, status: :created }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: { error: @user_file.errors.full_messages.join(', ') }, status: :unprocessable_entity }
      end
    end
  end

  def update
    respond_to do |format|
      if @user_file.update(user_file_params)
        format.html { redirect_to user_files_path, notice: "файл сохранен" }
      else
        format.html { render :edit, status: :unprocessable_entity }
      end
    end
  end

  def destroy
    @user_file.destroy!
    respond_to do |format|
      format.html { redirect_to user_files_path, notice: "файл удален" }
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
    if @user_file.folder.nil? || @user_file.folder.user != current_user
      redirect_to user_files_path, 
                 alert: "You are not authorized to perform this action.",
                 status: :forbidden
    end
  end
end
```
Обновим модель UserFile для обработки загрузки по URL:
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

    if file.image?
      file.variant(resize_to_limit: [200, 200]).processed
    
    elsif file.content_type == 'application/pdf' && file.previewable?
      file.preview(resize_to_limit: [200, 200], format: :jpg).processed
    end
    rescue ActiveStorage::FileNotFoundError, ActiveStorage::UnpreviewableError => e
        Rails.logger.error "Preview error: #{e.message}"
        nil
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

  def self.create_from_url(url, user, folder_id = nil)
    tempfile = Down.download(url)
    user_file = user.user_files.build(folder_id: folder_id)
    user_file.file.attach(io: tempfile, filename: File.basename(tempfile.path))
    user_file.save
    user_file
  ensure
    tempfile.close! if tempfile
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
