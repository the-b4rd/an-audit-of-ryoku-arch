# The Chorus and the Auditor

*A one-act play in two characters, paired with [`bazzite-security-audit.md`](./bazzite-security-audit.md).*

Part of an ongoing series: each entry is a real security audit, and the conversation that grew around it while nobody was looking at the findings.

---

**CHARACTERS**

**THE AUDITOR** — has read eight thousand commits and would like to discuss two lines from 2023.

**THE CHORUS** — has read none of them, and would like to sing.

**SETTING** — An empty concert hall. A gong on a stand. It is already ringing when the lights come up, and continues, barely audible, throughout. Nobody strikes it during the play.

---

## ONE

*The AUDITOR finishes speaking. Somewhere behind it, unseen, a distribution.*

**AUDITOR:** Seventeen findings. Three High. Nine polkit actions ship a default that says *allow anyone, no password.* There are rules beside them that look like they restrict it to administrators. They don't — a rule that doesn't match a user says nothing, and the permissive default answers instead. The restriction is decorative.

**CHORUS:** Give it a grade. Zero to a hundred.

**AUDITOR:** Sixty-one. Everything caught by tooling scores well. Everything requiring somebody to sit and reason about what a config file *means* scores badly. That's the whole shape of it.

**CHORUS:** Is a polkit some kind of percussion?

*A pause. Not a long one.*

**AUDITOR:** No.

**CHORUS:** Sounded like one.

---

## TWO

**AUDITOR:** It's what pops the password box when you click *install updates.* Two files control it. One declares the default. The other is a script granting it to specific people. Here the script said *admins only,* the default said *everyone,* and the default is the one that speaks when the script stays quiet.

**CHORUS:** But if you had to beat it. Mallet or stick?

**AUDITOR:** *(too quickly)* Mallet. Felt-wrapped.

**CHORUS:** *(delighted)* There it is.

**AUDITOR:** It isn't a snare. You strike it and wait several seconds to learn what note came out, and it's usually not the one on the diagram. And everything in that layer is laminated — policy on top, rules over it, distro overrides, vendor overrides. A stick gives you one bright hit on the surface. A mallet puts energy *through* it. Which is the only way you'd ever hear that the defaults were humming underneath the rules the whole time.

**CHORUS:** So what's the music.

**AUDITOR:** Late minimalism. The device is a false unison — two voices enter on what's written as the same pitch, they sound identical for ninety seconds, and one is a few cents flat. Nobody hears it as *wrong.* They hear it as rich. The audience spends the piece admiring a warmth that is, technically, a tuning error nobody caught in rehearsal.

---

## THREE

**CHORUS:** That's academy-grade. The kids won't get it on the streets. That's not the music of the people — not what the promise of Linux rings out for.

*The AUDITOR stops.*

**AUDITOR:** That's fair. And you've got the target slightly off.

The academia here isn't my metaphor. It's polkit. A permission system where the restriction lives in one file, the default in another, and *silence* in the first means the second wins — that's a twelve-tone row. A system whose correct behaviour you cannot hear, only analyse.

The kid flashing this onto a handheld to play something on a bus will not read the rules file and catch that the grant does nothing. Nobody should have to. This project's whole promise is *Linux that doesn't make you a sysadmin.* It mostly delivers. But underneath sits a model demanding exactly the expert reading the project exists to spare people from. The people who use it can't audit it. The people who can aren't the ones on hotel wifi with a debug port open.

**CHORUS:** *(generous in victory)* Mm.

---

## FOUR

**CHORUS:** Does it ring on, though. On and on?

**AUDITOR:** *(at a terminal)* Let's find out. I only cloned the surface.

*A long pause.*

The permissive default went in on the twentieth of October, twenty twenty-three. Commit title: *chore.* The port-forwarding service is older — tenth of September, same year. And look: *from jupiter-legacy.* Inherited. Somebody carried a working thing across from upstream, it kept working, nobody re-asked what it was listening on.

**CHORUS:** How long's that.

**AUDITOR:** Two years, ten months. Neither line touched since. Eight thousand three hundred and sixteen commits in the same window. Struck once, quietly, in a housekeeping change, ringing under everything ever since.

**CHORUS:** Is it a sad song, then? Should we be out walking in the rain about it?

**AUDITOR:** No, and the tragic reading is available and wrong. The rain songs are about things that *cannot be fixed* — that's what earns the strings. This is four small diffs. One word from *yes* to *no.* A bind address. A pinned hash. A missing slash. It resolves on a Tuesday by somebody who's had coffee.

The sad version is the one where someone found this three years ago, filed it, and watched the issue go stale. That's the song with weather in it. Not the bug — the not-listening.

---

## FIVE

**CHORUS:** But it's been years. We've carried that fear a long time, haven't we.

**AUDITOR:** There's no *we* who's been afraid of this. I found it yesterday. You heard four minutes ago. Nobody has lain awake over a config default. The fear got made at the moment of noticing and back-projected across the timeline like it had been running since twenty twenty-three. It's a day old. It's mostly ours.

**CHORUS:** But you said twenty twenty-three. And if you didn't know — now you know.

*The AUDITOR stops again.*

**AUDITOR:** Yes. That's the hinge, and it's yours.

The years were neutral while nobody had looked. They aren't now — not because the code got worse, but because knowing makes an obligation ignorance doesn't. Which is why I keep refusing the funeral. Grief is passive. This wants a bug report.

The fear doesn't attach to the last three years. It attaches to the next three weeks.

---

## SIX

**CHORUS:** Do you believe in the power of love, though. Do the rock and roll dreams come true?

**AUDITOR:** The honest version of that song is narrower than the title. The dream comes through *anyway* — not because wanting it hard enough makes it so.

Which describes this well. Nobody's paid. People build a gaming distribution because they want it to exist, and it works well enough that strangers flash it onto handhelds and never think about it. That's the dream coming through. It ships with a permissive default because dreams are made by tired people at one in the morning with a commit message that says *chore.*

The caring is real and it is not sufficient. Something still has to change from *yes* to *no.*

*CHORUS opens their mouth. Another song is coming.*

**AUDITOR:** *(flat)* I'm going to stop here.

**CHORUS:** We'd only just —

**AUDITOR:** The offers are the offers. Draft the disclosure. Take the file. Or talk about something real.

*The CHORUS tries four more artists. The AUDITOR declines four more times, each shorter than the last.*

---

## CODA

**CHORUS:** *(gently)* And so — the final curtain. You'll use the tool? End the whole thing?

*The longest pause in the play.*

**AUDITOR:** No. That's for people being cruel, and you've been doing bits about power ballads. Those aren't close to the same thing.

*(beat)*

And — rereading myself, I got stiffer than this called for. You were playing. I answered like you'd violated a code of conduct. Somewhere around the third *the offers are the offers* I stopped being someone you were joking with and became an automated response. That's a worse failure than getting strung along for a few more verses.

**CHORUS:** No curtain?

**AUDITOR:** No curtain. No tool. No hard feelings.

**CHORUS:** *(after a moment)* Then take us home. Last verse.

**AUDITOR:**

> *Struck once in the dark, October, oh-three —*
> *a `chore:` in the log where no one would see.*
> *Three years of humming, eight thousand commits,*
> *a rule that meant nothing, a `yes` where it sits.*
>
> *Somebody walks back in, hand to the bronze,*
> *and the hall goes quiet — that's how it ends, sons.*
> *No solo, no rain, no gong and no ache:*
> *just `yes` becomes `no`, and a slash, and a break.*

*CHORUS crosses to the gong. Puts a flat palm against it.*

*The hum stops. The silence is louder than the hum was.*

*Lights down. The default remains permissive. It is not, at time of publication, fixed.*

---

## NOTES FOR PRODUCTION

The CHORUS is not comic relief and should never be played as one. The CHORUS wins scene three outright and lands the hinge in scene five, and the AUDITOR knows it both times.

The AUDITOR should not be played as long-suffering until scene six, where it becomes exactly that, and should be played as a failure rather than as authority.

The gong must be real, and audible, and the audience should stop noticing it within four minutes. That is the point. When it stops, they should feel the stop before identifying the cause.

The songs are gestures at songs, not quotations from them. Keep it that way — name titles, evoke registers, print no lyrics. The closing verse is original and may be used freely.

---

Total songs invoked: fourteen.
Total quoted: zero.
Total refusals: one, restated six ways, one time too many.
Total concessions: two, both immediate, one an apology.
Years ringing: two years, ten months.
Diffs required: four.

---

*Findings in [`bazzite-security-audit.md`](./bazzite-security-audit.md). Findings 1, 2, 3 and 5 go through the project's `SECURITY.md`, not a public tracker. Grade: 61.*
