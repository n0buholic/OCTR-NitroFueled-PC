# Private CUP, Graphics, and Itemless evidence

I captured these screenshots during my own Linux host/guest testing.
They illustrate the feature work across development sessions; they are not
all captures of the final rebased commit.

| Screenshot | What it demonstrates |
| --- | --- |
| [Mode and race type](cup_selection.png) | ITEMLESS and CUP selected independently in a private lobby. |
| [Host and guest lobby](cup_lobby_display1.png) | Matching Velo CUP name, four-track preview, and CUP RACE - ITEMLESS subtitle. |
| [Private settings](cup_lobby_display2.png) | CUP/laps/difficulty on the host while the guest retains the selected CUP card. |
| [Track 2 results](cup_scoreboard1.png) | Matching race awards on both clients and NEXT RACE 3/4. |
| [Track 3 standings](cup_scoreboard2.png) | Matching cumulative scores, retained AI roster, and NEXT RACE 4/4. |
| [Track 4 results](cup_scoreboard3.png) | Host cumulative standings and guest per-race awards at different animation phases. |
| [Graphics selectors](new_graphic_setting_selector.png) | Native selectors and horizontal white help text at the lower left, during a race. |
| [Graphics entry](new_graphic_setting_gap_fix.png) | Graphics follows Controls when Extras is absent, without the old empty gap. |

The final-track image shows **Hasty 25 / n0buholic 21** in cumulative standings
on the left; **n0buholic +9 / Hasty +6** on the right are that race's awards,
not a second cumulative table. This is also the score case used by the podium
regression test. I separately confirmed that the corrected podium works and
that contact with players and AI in Itemless preserves reserves. Those checks
are manual results, not claims inferred from a still image.

Graphics screenshots precede upstream's change from Ultrawide to Aspect Ratio.
The PR preserves upstream's 16:9 / 21:9 / 32:9 choices. Top-edge selector
background clipping remains a reported visual limitation.
