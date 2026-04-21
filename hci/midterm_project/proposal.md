# Night City Block: An Ambient, Place-Based Social Awareness System for Intimate Friend Groups

**HCI / HAX Semester Project — Midterm Proposal**

---

## 1. Research Motivation

### 1.1 The Problem (ABT Framework)

We live in an era of unprecedented digital connectivity. Social media platforms, group chats, and notification systems give us constant access to information about the people in our lives — globally, over 5 billion people now use social media platforms (Roberts et al., 2025). **And** yet, a quiet but growing body of evidence suggests that this connectivity is not translating into felt closeness. A nine-year longitudinal study following nearly 7,000 adults found that both passive and active social media use were linked to increased feelings of loneliness over time — suggesting that the very platforms designed to connect people may deepen isolation rather than relieve it (Roberts et al., 2025). Crucially, this effect is most pronounced among people who use social media specifically to *maintain* close relationships — precisely the use case it promises to serve. **But** the dominant paradigms of social technology — feed-based broadcasting, reactive messaging, algorithmic curation of attention — are optimized for mass engagement, not for the subtle, low-effort awareness that characterizes being near someone you care about. Smartphone notifications alone have been shown to significantly increase inattention and reduce psychological well-being even when not acted upon (Kushlev & Dunn, 2016). Notifications demand; feeds compete; messages require a reply. **Therefore**, there is a meaningful design gap for systems that support presence-awareness and emotional connection among close friends without triggering the very attention burdens that make current platforms feel exhausting.

### 1.2 Why This Matters

This is not merely a UX inconvenience. The stakes of social disconnection are well-established and severe. Holt-Lunstad et al.'s landmark 2015 meta-analysis of over 3.4 million participants across 70 studies found that loneliness is associated with a 26% increased risk of early mortality, with social isolation linked to a 29% increase — effect sizes comparable to those of obesity and smoking. Their 2010 meta-analysis further demonstrated that being socially connected increases odds of survival by 50%, controlling for age and initial health status. Yet the modern urban lifestyle — characterized by geographic dispersal, smaller households, and delayed family formation — is structurally reducing the quantity and quality of close social contact (Holt-Lunstad et al., 2010).

The ambient texture of day-to-day life — the quiet knowledge that a friend is awake at 2am, or going through something heavy, or in a particularly good mood — has no meaningful counterpart in digital systems. Current platforms either broadcast too broadly (public feeds) or demand too explicitly (direct messages requiring a response). Awareness systems and calm technology have been studied in HCI for over two decades since Weiser and Brown (1996), but the majority of that work either targets workplace collaboration contexts (e.g., StudioBridge, CareNet) or focuses on elderly users and family communication. For the population of young adults navigating urban life, geographic dispersal, and emotionally significant but geographically distant friendships, ambient social connection in an expressive, low-demand register remains a critically underexplored design territory.

### 1.3 The Interaction Scenario

We are designing **Night City Block**, a shared digital urban environment where a small group of close friends (~5 people) each inhabit a personal "block" — rendered as a lit apartment window, shopfront, or rooftop in a living night-city scene. The city is not a social feed or a messaging interface. It is a collective ambient display: your block reflects you through the traces of a private daily diary (a drawing, a photo, a chosen color, a single word), and you discover your friends by wandering through theirs. Nobody sends anything directly. Connection happens through the city noticing you both.

The system is explicitly designed around calm technology principles: information flows to the periphery, not the center. You are not asked to respond. You do not receive notifications. You simply coexist — and the city, over time, tells a layered story of who was here, when, and in what state of mind.

### 1.4 Target Audience

The primary target users are **young adults in intimate friend groups (3–8 people)** who are geographically dispersed — living in different apartments, cities, or time zones — and who want lightweight, emotionally resonant awareness of each other without the pressure of explicit communication. Secondary contexts include long-distance couples and creative communities who value expressive over informational sharing.

---

## 2. Literature Review

### 2.1 Calm Technology and Peripheral Awareness

The foundational framework for this project is Weiser and Brown's (1996) theory of calm technology. They argue that well-designed technology should engage both the center and the periphery of attention, and — crucially — should move easily between the two. Their concept of *locatedness* is particularly relevant: a calm technology places us in a familiar environment in which the periphery is working well, allowing us to be "tuned in to what is happening around us" without cognitive overload. The dangling string — a physical indicator of network traffic requiring no interpretation — exemplifies the design ideal we are pursuing. Night City Block is built on this principle: the city is a peripheral display that can become a center of attention when the user chooses to explore it, but demands nothing in its resting state.

### 2.2 Tangible Bits and Physical–Digital Integration

Ishii and Ullmer's (1997) *Tangible Bits* vision proposes bridging the gap between the physical world and digital information through ambient media — light, sound, airflow, water — that leverages the periphery of human perception. Their distinction between *graspable media* (foreground bits manipulated actively) and *ambient media* (background bits available peripherally) maps directly onto our system design. The private diary functions as graspable media; the city atmosphere is ambient media. The optional physical artifact extension — a small lit building model on a desk, whose windows glow according to friends' activity — instantiates the Tangible Bits vision in a domestic, intimate register.

### 2.3 Ambient Awareness and Social Closeness

Scissors et al. (2016) provide empirical evidence that ambient awareness — the peripheral accumulation of fragmentary social information — can develop a coherent sense of knowing others even in the absence of direct one-to-one communication. In their study of Twitter users, participants reported moderate-to-high ambient awareness of people in their network, noting that this awareness increased perceptions of approachability. This supports our design premise that indirect, trace-based social information can create meaningful closeness without requiring active exchange. Importantly, their data also show that this effect works best in small, familiar networks — the exact social scale Night City Block targets.

### 2.4 Presence Displays and Physical Connectedness

Dey and Guzman (2006) designed Presence Displays — physical, peripheral ambient displays of online presence for close friends and family — and found, in a five-week field study, that these displays produced significantly better awareness of and connectedness to loved ones than standard graphical presence indicators. Their work demonstrates that the *form factor* matters: physically embodied ambient displays outperform screen-based equivalents for fostering felt connection. This motivates our attention to a possible physical artifact (a small illuminated building model) as a key design direction.

### 2.5 Awareness Systems and Affective Benefits

Markopoulos et al. (2004) introduce the concept of connectedness as an affective benefit of awareness systems, and develop the ABC-Q instrument to measure it. Their empirical work with the ASTRA system demonstrates that sharing experiences at the moment they happen, in a non-interrupting manner, can yield measurable affective benefits for intimate social networks. This is critical for our proposal: it suggests that the value we are designing for — felt closeness, emotional attunement — is not only conceptually plausible but empirically measurable.

### 2.6 Relatedness Technologies and Design Tensions

A growing literature, summarized in Hassenzahl et al. (2012) and more recently in Wenhart et al. (2025), identifies the core strategies through which HCI systems can foster experiences of relatedness: awareness, expressivity, physicalness, gift-giving, joint action, and shared memory. Night City Block consciously deploys awareness, expressivity, physicalness (via the artifact layer), and shared memory (accumulated city history). However, Hassenzahl et al. also identify a critical design tension between relatedness (feeling connected to others) and autonomy (feeling in control of one's own attention and self-expression). Current social platforms tend to optimize for relatedness at the cost of autonomy — constant pings, social obligation to respond, performance anxiety. Night City Block's architectural privacy model (where visibility is determined by spatial proximity in the city, not by a settings menu) is a direct response to this tension: the system is designed so that the user always retains control over how much of themselves is visible.

### 2.7 Knowledge Gaps This Project Addresses

The existing literature on ambient awareness systems has two significant gaps that this project addresses:

1. **The aesthetic and expressive dimension of ambient social systems is understudied.** Most existing awareness systems — Presence Displays, ASTRA, CareNet — focus on conveying presence and availability, not on expressive or artistic self-representation. They tell you *that* someone is home; they do not tell you anything about *who* that person is. Night City Block proposes diary-driven, artistically-generated ambient output as the primary signal, creating a richer and more emotionally resonant form of ambient presence.

2. **The spatial and architectural metaphor has not been systematically explored as a privacy and intimacy interface.** Privacy in existing systems is managed through settings and permission layers. We propose using *spatial architecture* — inside a room, a window, a doorstep, the street — as a naturalistic and intuitive privacy model, and study whether this lowers the cognitive and social cost of managing self-disclosure.

---

## 3. Research Objectives

1. To design and prototype an ambient social awareness system — Night City Block — that supports felt closeness among intimate friend groups without requiring explicit communication acts.
2. To investigate whether architecturally-encoded privacy (spatial proximity as a metaphor for disclosure level) is a viable and comfortable alternative to settings-based privacy management in ambient social systems.
3. To explore whether diary-driven, expressive ambient output generates richer and more emotionally meaningful awareness than presence/availability-based signals.

---

## 4. Research Questions

**RQ1:** How does ambient, expressive trace-based social information (diary-generated city elements) affect felt closeness and awareness among members of an intimate friend group, compared to a baseline of no ambient display?

**RQ2:** Does the architectural metaphor of spatial proximity (distance in the city corresponds to disclosure level) adequately and comfortably represent users' desired privacy boundaries, without requiring explicit privacy settings?

**RQ3:** What design elements within the Night City Block environment (e.g., weather as collective mood, street traces as social history, simultaneous presence indicators) are most salient to users in forming a sense of ambient social connection?

---

## 5. Proposed System Design

### 5.1 Core Concept

Each user owns a **block** in a shared digital night city. Their block reflects their inner state through a private daily diary: a drawing, photograph, mood color, single sentence, or voice fragment. These diary entries are never directly shared — instead, they *generate the atmosphere* of the user's block. A drawing becomes street art on their wall. A chosen color tints their window light. Silence dims their curtains to a single warm lamp.

The city exists between the blocks. Weather, street textures, and ambient elements emerge from the collective state of the group. Friends discover each other by wandering — nobody sends; nobody receives. The city notices, and the city shows.

### 5.2 Privacy as Architecture

| Space | Visibility |
|---|---|
| Interior room | Only the user |
| Window display | Close friends |
| Front doorstep | All friends |
| The street | Everyone in the city |
| A back alley | One specific person only |

Privacy is not configured in a settings menu. It is encoded spatially, in the physical logic of the city.

### 5.3 Optional Physical Artifact

A small illuminated building model sits on the user's desk. Its windows glow with the ambient colors of friends' moods. It requires no screen time; it lives in the peripheral vision of the workspace.

---

## 6. Methodology (Planned)

This midterm proposal focuses on motivation, literature, and research objectives. The full methodology will be developed in subsequent milestones. At present, we anticipate:

- **Formative study:** Semi-structured interviews with young adults (aged 20–30) about their existing practices around ambient social awareness and closeness — what do they notice, what do they miss, what feels exhausting?
- **Prototype development:** A functional prototype of the Night City Block environment, with a small cohort of 4–6 participants.
- **Field deployment study (3–4 weeks):** Participants use the system with their actual friend group; we collect diary entries, usage logs, and periodic experience sampling.
- **Post-study interviews:** Qualitative inquiry into felt closeness, privacy comfort, and the salience of specific design elements.

---

## 7. Prof. McGee's 8 Questions — Self-Assessment

| Question | Our Response |
|---|---|
| **Research problem/question** | How can ambient, expressive, trace-based social systems support felt closeness in intimate friend groups without explicit communication demands? |
| **Is it significant?** | Yes — the tension between digital connectivity and felt loneliness is a documented social phenomenon; ambient awareness systems for intimate, expressive social contexts are underexplored. |
| **Contribution to knowledge** | A design prototype and empirical findings on (a) expressive ambient awareness vs. presence-only awareness, and (b) architectural vs. settings-based privacy as interaction design. |
| **Is the contribution vital?** | We believe so — the study outcomes could inform design of a whole class of calm, intimate social systems. |
| **Originality** | No existing system combines: night-city spatial metaphor + diary-to-atmosphere generation + architectural privacy model + physical artifact layer. |
| **How readers know it's original** | Literature review demonstrates the gaps; no known system combines all four design elements. |
| **Validity** | Mixed-methods field study with measurable affective outcomes (connectedness, using instruments from prior work such as ABC-Q) and qualitative depth. |
| **Disciplinary fit** | Core HCI / ubicomp / calm technology — with contributions relevant to CSCW and design research. |

---

## References

1. Weiser, M., & Brown, J. S. (1996). *The Coming Age of Calm Technology.* Xerox PARC. Retrieved from https://calmtech.com/papers/coming-age-calm-technology

2. Ishii, H., & Ullmer, B. (1997). Tangible bits: towards seamless interfaces between people, bits and atoms. *Proceedings of the ACM SIGCHI Conference on Human Factors in Computing Systems*, 234–241. https://doi.org/10.1145/258549.258715

3. Scissors, L., Burke, M., & Wengrovitz, S. (2016). What's in a Like? Attitudes and behaviors around receiving Likes on Facebook. In *Proceedings of CSCW 2016.* *(See also: Scissors et al., 2016, on ambient awareness — PMC4853799.)*

4. Dey, A. K., & Guzman, E. S. (2006). From awareness to connectedness: The design and deployment of presence displays. *Proceedings of the ACM CHI Conference on Human Factors in Computing Systems.* https://doi.org/10.1145/1124772.1124828

5. Markopoulos, P., van Baren, J., IJsselsteijn, W., de Ruyter, B., & Farshchian, B. (2004). Connecting the family with awareness systems. *Personal and Ubiquitous Computing*, 8(6), 405–414. https://doi.org/10.1007/s00779-004-0293-2

6. Hassenzahl, M., Heidecker, S., Eckoldt, K., Diefenbach, S., & Hillmann, U. (2012). All you need is love: Current strategies of mediating intimate relationships through technology. *ACM Transactions on Computer-Human Interaction (TOCHI)*, 19(4), 1–19.

7. Consolvo, S., Roessler, P., & Shelton, B. E. (2004). The CareNet display: Lessons learned from an in-home evaluation of an ambient display. *UbiComp 2004*, LNCS 3205, 1–17.

8. Bhowmick, P., & Stolterman, E. (2023). Exploring tangible user interface design for social connection among older adults: A preliminary review. *Extended Abstracts of CHI 2023.* https://doi.org/10.1145/3544549.3585722

9. Holt-Lunstad, J., Smith, T. B., & Layton, J. B. (2010). Social relationships and mortality risk: A meta-analytic review. *PLOS Medicine*, 7(7), e1000316. https://doi.org/10.1371/journal.pmed.1000316

10. Holt-Lunstad, J., Smith, T. B., Baker, M., Harris, T., & Stephenson, D. (2015). Loneliness and social isolation as risk factors for mortality: A meta-analytic review. *Perspectives on Psychological Science*, 10(2), 227–237. https://doi.org/10.1177/1745691614568352

11. Roberts, J. A., Young, P., & David, M. E. (2025). The epidemic of loneliness: A nine-year longitudinal study of the impact of passive and active social media use on loneliness. *Personality and Social Psychology Bulletin.* https://doi.org/10.1177/01461672241290235

12. Kushlev, K., & Dunn, E. W. (2016). "Silence your phones": Smartphone notifications increase inattention and hyperactivity symptoms. *Proceedings of the ACM CHI Conference on Human Factors in Computing Systems*, 1011–1020. https://doi.org/10.1145/2858036.2858359