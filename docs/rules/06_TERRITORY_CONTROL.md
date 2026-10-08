# Феодали — Приєднання областей: детальна специфікація V1

[Основні правила](../01_GAME_RULES_V1.md). Усі правила нижче збережені без змін із початкового документа.

# 27. Annexation Neutral Region

Annexation progress для Player може йти, якщо одночасно:

- Neutral Defense == 0;
- Player має в Region active Camp-presence; Regrouping рахується так само, як Camp;
- є хоча б одна valid-connected adjacent Owned Region цього Player;
- немає active Founding;
- немає blocking foreign Camp-presence: foreign Camp, Regrouping або entered-for-Camp Army.

Foreign pure Transit не pause-ить Annexation progress, навіть якщо через цей Transit існує CombatSituation.

Поява blocking foreign Camp/Camp-bound presence pause-ить progress, але не скидає його. Якщо власний Camp episode повністю завершується до Annexation, накопичений control progress цього episode втрачається.

Досягнення required control time не виконує Annexation автоматично. Region лише стає ready.

При manual Annexation Player обирає конкретний **незаблокований** Castle; Annexation до заблокованого Castle заборонений. Потрібні:

- valid territorial connection до цього Castle;
- вільна Governor's House Capacity;
- у Camp має бути хоча б один Knight, чия Army має Camp-presence і чий home Castle — саме обраний Castle;
- відсутність blocking foreign Camp/Camp-bound presence;
- відсутність active Founding.

Annexation не має окремої одноразової resource cost у V1.

Неважливо, хто саме знищив Neutral Defense; право на control progress визначається поточною presence та connectivity.

---

# 28. Annexation Occupied Region

**Castle Region не може бути Annexed** навіть за Occupation. Це стосується Capital та інших Castle Region. Так само до заблокованого Castle не можна Annex нові Region.

Current occupier може Annex Occupied Region за тією самою control-progress логікою, але Neutral Defense для цього не потрібна.

Progress потребує власної Camp/Regrouping presence occupier, valid-connected adjacent Owned Region та відсутності blocking foreign Camp/Camp-bound presence. Pure Transit не блокує.

Після накопичення required time Annexation виконується окремою user action з вибором Castle та перевірками connection, Governor Capacity і наявності в Camp Knight з home Castle, що дорівнює обраному Castle.

До завершення Annexation формальний owner лишається owner і продовжує Region upkeep.

Після Annexation ownership переходить occupier, Region приєднується до обраного Castle, а territorial connectivity колишнього owner перераховується. Region, які через остаточну втрату bridge більше не мають шляху до свого Castle, стають Neutral за правилами розділу 3.

---

