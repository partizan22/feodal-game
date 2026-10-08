# Феодали — Заснування замків: детальна специфікація V1

[Основні правила](../01_GAME_RULES_V1.md). Усі правила нижче збережені без змін із початкового документа.

# 31. Заснування нового Castle

У V1 новий Castle можна заснувати **тільки у Neutral Region**. Початковий Castle кожного гравця — його Capital. У V1 столицю не можна перенести; нові Castle не є Capital. Лише на Capital Region поширюються заборони атаки та чужого Transit. Заснування Castle у власній Owned/Annexed Region заборонене.

Для Founding не потрібні contiguity, Governor Capacity або попередній Annexation progress, але Neutral Defense має бути 0.

Founder — Knight без Soldier, фізично присутній у Camp цієї Region. Regrouping рахується Camp-presence: він не забороняє старт Founding і не pause-ить progress сам по собі.

На Start:

- active Founding у Region не повинно бути;
- foreign blocking Camp-presence не повинна існувати;
- Player сплачує local founding cost із home Castle founder Knight і global cost із Player;
- задається name нового Castle.

Foreign Camp, Regrouping або entered-for-Camp presence після Start pause-ить Founding progress. Pure Transit не pause-ить його, навіть якщо через Transit існує CombatSituation.

Founder повинен залишатися живим і залишатися у потрібному Camp увесь процес. Вимога нульової кількості Soldier перевіряється при Start: змінити склад Unit можна лише у home Castle, тож під час Founding Founder не може отримати Soldier. Якщо Founder залишає Region/Camp або гине, Founding скасовується без refund.

У Neutral Region active Founding і Annexation взаємовиключні.

До completion Founding діють правила Neutral Region; після completion — звичайні правила Owned Region для не столичної Castle Region. Якщо до completion чужа Army вже фізично ввійшла в Region із метою Transit, вона завершує розпочатий Transit без CombatSituation та без блокування Castle. Лише запланований Route такого винятку не створює.

Після completion Neutral Region стає Castle Region нового Castle. Існуючі City, `wealth`, `active_wealth_ratio`, ResourceSite та їх levels зберігаються; active ResourceSiteUpgrade продовжуються без reset. Neutral Defense після переходу Region у Castle Region більше не має gameplay-функції. Створюються Warehouse 1, Granary 1 і Palace 1. Founder змінює home Castle на новий і займає початковий Palace slot; додатковий Knight через цей стартовий slot не генерується. У старому home Castle founder-а звільнений Palace slot запускає звичайний KnightReplacement mechanism так само, як slot після загибелі Knight. Кожний Palace slot може перебувати лише в одному стані: зайнятий living Knight, зарезервований ready unnamed Knight або зарезервований active/pending KnightReplacement. Transfer Founder звільняє рівно один slot, який резервується для його replacement; наявні ready unnamed entries не створюють додаткових slots.

---

