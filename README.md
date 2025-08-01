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
```ruby
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

  def high_quality_preview
    return unless file.attached?
    
    begin
      if file.image?
        file.variant(
          resize_to_limit: [800, 800],
          saver: {
            quality: 90,
            strip: true,
            interlace: 'JPEG'
          }
        ).processed
      elsif file.content_type == 'application/pdf'
        file.preview(
          resize_to_limit: [800, 800],
          format: :jpg,
          saver: {
            quality: 90,
            strip: true
          }
        ).processed
      end
    rescue ActiveStorage::FileNotFoundError, ActiveStorage::InvariableError => e
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
end
```
Контроллер коллекций
```ruby
class CollectionsController < ApplicationController
  before_action :authenticate_user!
  before_action :set_collection, only: %i[ show edit update destroy ]
  before_action :authorize_user, only: %i[ edit update destroy ]

  # GET /collections or /collections.json
  def index
    @collections = Collection.all
  end

  # GET /collections/1 or /collections/1.json
  def show
  end

  # GET /collections/new
  def new
    @collection = Collection.new
  end

  # GET /collections/1/edit
  def edit
  end

  # POST /collections or /collections.json
  def create
    @collection = Collection.new(collection_params)

    respond_to do |format|
      if @collection.save
        format.html { redirect_to @collection, notice: "Collection was successfully created." }
        format.json { render :show, status: :created, location: @collection }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: @collection.errors, status: :unprocessable_entity }
      end
    end
  end

  # PATCH/PUT /collections/1 or /collections/1.json
  def update
    respond_to do |format|
      if @collection.update(collection_params)
        format.html { redirect_to @collection, notice: "Collection was successfully updated." }
        format.json { render :show, status: :ok, location: @collection }
      else
        format.html { render :edit, status: :unprocessable_entity }
        format.json { render json: @collection.errors, status: :unprocessable_entity }
      end
    end
  end

  # DELETE /collections/1 or /collections/1.json
  def destroy
    @collection.destroy!

    respond_to do |format|
      format.html { redirect_to collections_path, status: :see_other, notice: "Collection was successfully destroyed." }
      format.json { head :no_content }
    end
  end

  private
    # Use callbacks to share common setup or constraints between actions.
    def set_collection
      @collection = Collection.find(params[:id])
    end

    # Only allow a list of trusted parameters through.
    def collection_params
      params.require(:collection).permit(:name, :user_id, :is_public)
    end

    def authorize_user
      unless @collection.user == current_user
        redirect_to collections_path, alert: "You are not authorized to perform this action."
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

  # GET /user_files or /user_files.json
  def index
    @user_files = if params[:mine] == 'true'
                    current_user.user_files
                  else
                    UserFile.visible_to(current_user)
                  end
                  .includes(:collection, file_attachment: :blob)
  end

  # GET /user_files/1 or /user_files/1.json
  def show
  end

  # GET /user_files/new
  def new
    @user_file = UserFile.new
    @collections = current_user.collections
  end

  # GET /user_files/1/edit
  def edit
    @collections = current_user.collections
  end

  # POST /user_files or /user_files.json
  def create
    @user_file = current_user.user_files.build(user_file_params)
    @collections = current_user.collections

    respond_to do |format|
      if @user_file.save
        format.html { redirect_to user_file_url(@user_file), notice: "User file was successfully created." }
        format.json { render :show, status: :created, location: @user_file }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: @user_file.errors, status: :unprocessable_entity }
      end
    end
  end

  # PATCH/PUT /user_files/1 or /user_files/1.json
  def update
    respond_to do |format|
      if @user_file.update(user_file_params)
        format.html { redirect_to user_file_url(@user_file), notice: "User file was successfully updated." }
        format.json { render :show, status: :ok, location: @user_file }
      else
        format.html { render :edit, status: :unprocessable_entity }
        format.json { render json: @user_file.errors, status: :unprocessable_entity }
      end
    end
  end

  # DELETE /user_files/1 or /user_files/1.json
  def destroy
    @user_file.destroy!

    respond_to do |format|
      format.html { redirect_to user_files_url, notice: "User file was successfully destroyed." }
      format.json { head :no_content }
    end
  end

  private
    # Use callbacks to share common setup or constraints between actions.
    def set_user_file
      @user_file = UserFile.find(params[:id])
    end

    # Only allow a list of trusted parameters through.
    def user_file_params
      params.require(:user_file).permit(:name, :collection_id, :file)
    end

    def authorize_user
      unless @user_file.collection.user == current_user
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
    <%= f.input :name %>
    <%= f.association :collection, collection: current_user.collections %>
    
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
            <%= image_tag preview, class: 'card-img-top img-fluid' %>
          <% else %>
            <div class="card-img-top bg-light d-flex align-items-center justify-content-center" style="height: 200px;">
              <i class="<%= user_file.icon_for_file %>" style="font-size: 3rem;"></i>
            </div>
          <% end %>
        <% else %>
          <div class="card-img-top bg-light d-flex align-items-center justify-content-center" style="height: 200px;">
            <i class="bi bi-file-earmark" style="font-size: 3rem;"></i>
          </div>
        <% end %>

        <div class="card-body">
          <h5 class="card-title"><%= user_file.name %></h5>
          <p class="card-text">
            <small class="text-muted">
              Collection: <%= user_file.collection.name %>
              <% if user_file.collection.is_public? %>
                <span class="badge bg-success ms-2">Public</span>
              <% else %>
                <span class="badge bg-secondary ms-2">Private</span>
              <% end %>
            </small>
          </p>
        </div>

        <div class="card-footer bg-white">
          <%= link_to 'Show', user_file, class: 'btn btn-sm btn-outline-primary' %>
          <%= link_to 'Edit', edit_user_file_path(user_file), class: 'btn btn-sm btn-outline-secondary' %>
          <%= link_to 'Delete', user_file, 
                      method: :delete, 
                      data: { confirm: 'Are you sure?' }, 
                      class: 'btn btn-sm btn-outline-danger' %>
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
  resources :collections
  resources :user_files
  
  root to: "collections#index"
end
```
