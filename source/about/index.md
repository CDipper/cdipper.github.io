---
title: About
layout: page
---

<style>
.about-wrap { max-width: 680px; margin: 0 auto; text-align: center; }
.about-avatar {
  width: 120px; height: 120px; border-radius: 50%;
  object-fit: cover; border: 3px solid #2bbc8a;
  box-shadow: 0 4px 18px rgba(0,0,0,.45);
  margin-bottom: 18px;
}
.about-name { font-size: 1.8rem; color: #eeeeee; margin: 0 0 6px; font-weight: 600; }
.about-bio { color: #908d8d; margin: 0 0 30px; font-size: .95rem; }
.about-cards { display: flex; flex-direction: column; gap: 14px; text-align: left; }
.about-card {
  background: #25272a; border: 1px solid #333; border-left: 3px solid #2bbc8a;
  border-radius: 8px; padding: 14px 18px;
}
.about-card h3 {
  margin: 0 0 6px; font-size: .82rem; color: #2bbc8a;
  font-weight: 600; letter-spacing: 1px; text-transform: uppercase;
}
.about-card p { margin: 0; color: #c9cacc; font-size: .92rem; line-height: 1.6; }
.about-card a { color: #d480aa; text-decoration: none; }
.about-card a:hover { text-decoration: underline; }
.about-links { margin-top: 30px; display: flex; gap: 12px; justify-content: center; flex-wrap: wrap; }
.about-link {
  display: inline-block; padding: 8px 20px; border: 1px solid #2bbc8a;
  border-radius: 20px; color: #2bbc8a; text-decoration: none; font-size: .85rem;
  transition: all .2s ease;
}
.about-link:hover { background: #2bbc8a; color: #1d1f21; }
</style>

<div class="about-wrap">
  <img src="/assets/config/avatar.jpg" alt="avatar" class="about-avatar">
  <h1 class="about-name">CDipp3r</h1>
  <p class="about-bio">Second-year graduate student · Offensive Security</p>

  <div class="about-cards">
    <div class="about-card">
      <h3>Email</h3>
      <p><a id="about-email" data-mail="MTk2NDY4MjY0MEBxcS5jb20=">decoding…</a></p>
    </div>
    <div class="about-card">
      <h3>Research Interests</h3>
      <p>Windows Security · Offensive Security · Reverse Engineering · Red Team · Web Security</p>
    </div>
    <div class="about-card">
      <h3>Identity</h3>
      <p>Second-year graduate student</p>
    </div>
  </div>

  <div class="about-links">
    <a class="about-link" href="https://github.com/CDipper" target="_blank" rel="noopener">GitHub</a>
    <a class="about-link" data-mail="MTk2NDY4MjY0MEBxcS5jb20=">Email</a>
  </div>
</div>

<script>
  (function () {
    document.querySelectorAll('[data-mail]').forEach(function (el) {
      var mail = atob(el.getAttribute('data-mail'));
      el.href = 'mailto:' + mail;
      if (el.textContent.trim() === 'decoding…') el.textContent = mail;
    });
  })();
</script>
