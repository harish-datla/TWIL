# A short history

Why does an algorithm a bot-contest player named half as a joke in 2018 have a decade of academic papers behind it? Because general game playing and game theory had found and analyzed the same idea, under another name, years earlier.
Timeline: from UCB1 (2002) to Simultaneous AlphaZero (2025)

In short: UCB1 (2002)[1] became UCT, its tree-search form, in 2006[2]. CadiaPlayer kept separate statistics per player for simultaneous moves at the 2007 General Game Playing competition[3], where programs play games whose rules they only receive at the start. From 2009 to 2016 researchers analyzed the SM-MCTS (simultaneous-move MCTS) variants formally[22][5][8]. CodinGame players adopted the idea under the "Smitsimax" name in 2018[9], and neural networks joined it in 2024 and 2025[10][11]. The algorithm MSmits named maps closely to Decoupled UCT (DUCT), with one implementation difference: Smitsimax keeps an entirely separate tree per agent, where DUCT keeps one shared tree whose nodes hold per-player statistics. The Ludii general game system's example DUCT agent is a compact reference implementation of the academic form[17].
