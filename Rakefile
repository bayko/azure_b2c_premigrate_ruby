require "bundler/setup"
require 'dotenv/load'
require 'httparty'

def initialize_graph
  log_step 'start', 'initialize_graph'
  @tenant = ENV['AZURE_TENANT']
  #@tenant = 'quarkbiz.onmicrosoft.com' # https://mytenant.onmicrosoft.com
  # clientID and secret for the migration application in b2c with access to the ms graph api
  # this should be different than the b2c clientID and secret for omniauth strategy
  @client_id      = ENV['AZURE_MIGRATION_ID']
  @client_secret  = ENV['AZURE_MIGRATION_SECRET']
  # this is the name of the special attribute in b2c which will tell the system users require migration from legacy auth
  @migration_flag = ENV['AZURE_MIGRATION_FLAG']
  # the object id for the built in b2c-extensions-app
  @extension_oid = ENV['AZURE_EXTENSION_OBJECT_ID']
  # we need to use the ms graph api to interact with users
  @resource = "https://graph.microsoft.com"
end


# we need an access token that has permissions to read/write on the directory and application
def get_auth_token
  log_step "start", "get_auth_token"
  login_url = "https://login.microsoft.com"
  token_url = "#{login_url}/#{@tenant}/oauth2/token?api-version=1.0"
  body = {
    client_id: @client_id,
    client_secret: @client_secret,
    grant_type: 'client_credentials',
    resource: @resource
  }
  response = HTTParty.post "#{token_url}", body: body
  @access_token = response['access_token']
  log_step 'auth_token', @access_token
end


def log_step x, s
  puts "**********"
  puts "#{x}: #{s}"
end


task :register_extension => :environment do
  initialize_graph()
  get_auth_token()
  
  def register_custom_extension
    # register the migration attribute in B2C extensions app so we can flag legacy users during pre-migration
    # only needs to be done once
    post_url = "#{@resource}/v1.0/applications/#{@extension_oid}/extensionProperties"
    attribute = {}
    attribute["name"] = 'requiresMigration'
    attribute["dataType"] = 'Boolean'
    attribute["targetObjects"] = []
    attribute["targetObjects"].push "User"
    body = attribute.to_json
    response = HTTParty.post "#{post_url}",
      body: body,
      headers: {
        "Authorization" => "#{@access_token}",
        "Content-Type" => "application/json",
        "ContentLength" => "#{body.length}"
      }
    result = JSON.parse response.body
    log_step 'register_cusom_extension', result
  end
end


# this task is just for checking/testing a individual user to confirm if the flag is set correctly
task :check_migration_flag => :environment do
  initialize_graph()
  get_auth_token()
  
  def check_migration_flag user
    # check if the migration flag is set to true or false
    # it should be true before the users first logon and false after
    get_url = "#{@resource}/v1.0/users/#{user.b2c_id}?$select=#{@migration_flag}"
    response = HTTParty.get "#{get_url}",
      headers: {
        "Authorization" => "#{@access_token}",
        "Content-Type" => "application/json"
      }
    result = JSON.parse response.body
    log_step 'check_migration_flag', result
  end
  
  # testing single user migrate only
  # check_migration_flag(User.find(2))

end


# this task is to pre-migrate all legacy users in the app to b2c following the seamless strategy
# https://github.com/azure-ad-b2c/user-migration/tree/master/seamless-account-migration
task :premigrate_legacy_users => :environment do
  initialize_graph()
  get_auth_token()
  
  # once authenticated we need to call a different endpoint to create the user in b2c
  # using a special attribute so our b2c policy can see they require migration during next login
  def create_b2c_user user, count
    # refresh the graph token after every 100 users
    if (count%100) == 0
      get_auth_token()
    end

    # we need to define the required fields for b2c to generate a user with a dummy password
    # when the user first logs in through b2c the migration flag will query the rails app to authenticate before migrating
    u = {}
    u["displayName"] = user.email
    u["identities"] = [{}]
    u["identities"][0]["signInType"] = 'emailAddress'
    u["identities"][0]["issuer"] = ENV['AZURE_TENANT']
    u["identities"][0]["issuerAssignedId"] = user.email
    u["passwordProfile"] = {}
    u["passwordProfile"]["password"] = "#{SecureRandom.hex}!@A"
    u["passwordProfile"]["forceChangePasswordNextSignIn"] = false
    u["passwordPolicies"] = "DisablePasswordExpiration"
    u["#{@migration_flag}"] = true

    # create the user in b2c flagged for a required migration
    body = u.to_json
    post_url = "#{@resource}/v1.0/users"
    
    response = HTTParty.post "#{post_url}",
      body: body,
      headers: {
        "Authorization" => "#{@access_token}",
        "Content-Type" => "application/json",
        "ContentLength" => "#{body.length}"
      }
    result = JSON.parse response.body

    if result['error']
      log_step 'create_b2c_user failed', result['error']['message']
    else
      log_step 'create_b2c_user success', result['userPrincipalName']
      # record each users returned b2c object id in the database for authorization going forward
      user.update_columns({
        b2c_id: result['id']
      })
    end
  end

  # Insert rails logic here to grab a selection of users you want to migrate

  # users = User.where({
  #   b2c_id: nil
  # })

  # migrated_count = 0
  # users.each do |user|
  #   create_b2c_user user, migrated_count
  #   migrated_count += 1
  # end

end 


