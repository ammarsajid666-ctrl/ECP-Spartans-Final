ECP Spartans multi-page website

Upload this entire folder to Vercel. index.html is the Home page.
Pages: fixtures.html, finances.html, contributions.html, team-stats.html,
player-stats.html, admin.html.
Hero image: ecp-spartans-hero.jpeg

Team Stats and Player Stats were fixed to read the fixture schedule from
the same Supabase key ('matches') that Fixtures actually saves to — they
previously pointed at a key ('scheduleMatches') nothing ever wrote to, so
their schedule/results sections would have looked permanently empty.

ACCESS GATE
Every page now opens with a full-screen gate before showing any content:
  - Admin logs in with email/password (or a magic email link) using the
    Supabase Auth account that matches ADMIN_EMAIL in the script.
  - First-time setup: use the "Admin setup" tab to create that account
    (only the configured admin email is allowed to sign up).
  - Registered players use the "Player sign-up" tab to create their own
    email/password login (see PLAYER LOGINS below).
  - Anyone else taps "Continue as Guest" to browse fixtures, finances,
    and contributions read-only. Guests cannot add or edit data.
The gate re-appears on every page load until a real login session or the
guest choice is present in the browser (localStorage + Supabase session),
so a visitor landing on any page for the first time sees only the login
screen first, exactly like a login-gated app.

PLAYER LOGINS & SELF-SERVICE AVAILABILITY (new)
Every player can now have their own real login, separate from the single
admin account:
  1. Admin adds the player on the Team Sheet (admin.html), now with an
     Email field alongside Name and Phone. The email must be entered
     before that player can register.
  2. Either the player self-registers (the "Player sign-up" tab on the
     gate screen, same email + a password, min 6 chars — needs Supabase's
     "Confirm email" setting off or they'll never get the confirmation
     email), OR the admin creates the account directly in Supabase
     Dashboard -> Authentication -> Users -> Add user, with "Auto Confirm
     User" checked, and shares the email/password with the player
     (e.g. over WhatsApp). Either way they log in through the normal
     "Log in" tab. The email-on-Team-Sheet check is a friendly
     client-side check, not the real security boundary — see below.
  3. Once logged in, a player sees FIXTURES ONLY — match details and the
     availability roster, not Finances or Contributions. The Finances/
     Contributions nav links and the Home page's financial cards
     (expenses, contributions received, balance, Season Summary PDF) are
     hidden from players, and typing the finances.html/contributions.html
     URL directly redirects them back to Fixtures. Within a match's own
     detail popup, the "To your team" reminder section (which lists every
     player's phone number as a WhatsApp send button), "To opponent
     captain", and "Expenses for this match" are also hidden from
     players — those are admin actions (sending reminders, entering
     expenses) and expose other players' contact info, so a player only
     sees the match info and the availability roster there, nothing else.
     Guests still see fixtures, finances, and contributions, same as
     before — this restriction only applies to logged-in player accounts.
  3a. Home now opens with a "Next Match" card (index.html only, visible to
      everyone — admin, players, and guests) right at the top of Club
      Overview: the match details (opponent, date) on top, and directly
      under it the list of players who've confirmed Yes for that match,
      plus a count of who's not available and who hasn't replied yet.
      It's read-only here — a quick "who's actually playing" summary
      that updates itself the moment anyone marks their availability from
      any of the three places below (the Home quick card, a match's own
      details, or the Admin panel), no page reload needed.
  3a-i. Right under that, players (only) also get a "Your Availability"
      card, showing the same next match with three quick buttons (Yes,
      I'm in / Can't make it / Not sure yet) — a shortcut so a player
      doesn't have to open the match's full details just to mark
      themselves. It writes to the same availability_responses table as
      the roster inside a match's details, so marking from either place
      stays in sync, and also updates the Team confirmed-names card above
      it right away.
  3b. The Admin page (admin.html) now has a "Player Availability" panel
      above the Team Sheet, showing the whole roster's Yes/No/pending
      status for the next upcoming match without needing to open that
      match's details — so wherever a player marked themselves (the
      match popup or the Home card), admin sees it reflected there right
      away. Admin can also mark for any player from this panel, same as
      from inside a match's details. Above the player list, a "Team (N):
      Name1, Name2, ..." line shows the confirmed squad for that match at
      a glance (everyone who said Yes), plus a count of who's still
      pending or said no — the same summary style already used inside a
      match's own details popup.
  3c. Every place a player or admin can mark availability (the Home quick
      card, the roster inside a match's details, and the admin panel) now
      shows a small "Saving..." / "✓ Saved" / error line right under the
      buttons the moment you click. Previously a click that didn't change
      which button was already selected (e.g. tapping "Yes" again when you
      were already marked Yes) looked like nothing happened — the buttons
      only physically change when the status changes, so there was no
      other feedback that the click was received and saved. This line
      confirms every click regardless of whether the visual state changes,
      and shows the actual error message inline if the save fails (instead
      of only in the small status line near the login/logout button at the
      top of the page, which is easy to miss if you're scrolled down).
     Note this is
     a UI-level restriction, not a data-level one: because finances and
     contributions are read publicly (same as fixtures) so guests can
     view them, the underlying numbers are still fetched into the page's
     JavaScript for everyone including players — a player poking at
     browser dev tools could still find them there. If that data needs
     to be truly hidden from players (not just from the page they see),
     that requires a bigger change (splitting who can read what at the
     database level) — ask if you want that.
  4. Opening any match's details shows the full availability roster
     (everyone's Yes / No / pending, same as admin sees), and the
     player's own row has working Yes / No / ? buttons so they can mark
     themselves directly. Every other player's row is still read-only to
     them.
  5. Admin keeps full control too — the admin's own view of the roster
     still has every row editable, exactly as before, and admin still
     sees Finances/Contributions normally.
Real security is enforced at the database level, not in the page's
JavaScript: availability responses live in a new Supabase table,
availability_responses, with Row Level Security policies that only let
a signed-in user write a row where the row's player_email matches their
own login email (or write anything at all if they're the admin email).
That means even if someone bypassed the UI, they could still only ever
write availability for their own email address. Run the SQL in
availability-table-setup.sql once (Supabase Dashboard -> SQL Editor) to
create this table and its policies — the feature won't work until that's
been run. Reading the roster is public (same visibility as fixtures),
matching the "players can see the full team roster" requirement.

SEASON SUMMARY REPORT (new)
Home now has a "Season Summary Report" card under Club Overview. It builds
a single PDF (via jsPDF, same library Player Stats already uses) with:
  - Match record: completed matches, won/lost/tied/no-result, win %
    (read from the 'matchResults' key that Team Stats saves to)
  - Financial position: contributions received/expected, expenses, balance
  - Top 5 run scorers and top 5 wicket takers (from the 'matchStats' key
    that Player Stats saves to)
  - Players with a pending contribution balance
  - Next 6 upcoming fixtures
Available to anyone (guest or admin) since it only reads existing data —
it doesn't require Player Stats or Team Stats to have been filled in;
sections with no data yet print a plain "nothing recorded" line instead
of failing. Only index.html changed for this — the other pages are
untouched.
