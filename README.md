\documentclass{article}
\usepackage{graphicx} % Required for inserting images

\title{rapport}
\author{Sabri Malak}
\date{November 2026}

\begin{document}

\maketitle

\section{Introduction}
\documentclass[11pt,a4paper]{article}

\usepackage[utf8]{inputenc}
\usepackage[T1]{fontenc}
\usepackage[french]{babel}
\usepackage{lmodern}
\usepackage[margin=2.5cm]{geometry}
\usepackage{amsmath,amssymb}
\usepackage{booktabs}
\usepackage{graphicx}
\usepackage{float}
\usepackage{hyperref}
\usepackage{microtype}

\hypersetup{colorlinks=true,linkcolor=black,urlcolor=blue}

% Insère une figure seulement si le fichier existe (évite une erreur à la compilation)
\newcommand{\fig}[3]{%
  \IfFileExists{#1}{%
    \begin{figure}[H]\centering
      \includegraphics[width=#3\textwidth]{#1}
      \caption{#2}
    \end{figure}}{%
    \par\noindent\fbox{\parbox{0.95\textwidth}{\textit{Figure à insérer : \detokenize{#1}}}}\par}}

% Valeur à compléter, bien visible
\newcommand{\XX}[1]{\ifmmode\text{\textbf{[#1]}}\else\textbf{[#1]}\fi}

\title{\textbf{F8 : Battre la marche aléatoire}\\[2mm]
\large Prévision du signe du rendement quotidien du S\&P~500\\ par régression logistique}
\author{IBRAHIM EL-MERZOUKI \\ Code Apogée : 25007743 \quad Groupe : \XX{groupe} \quad Filière : \XX{filière}}
\date{17 novembre 2026}

\begin{document}
\maketitle

\begin{center}
\textbf{Contrôle continu, Partie B : projet de science des données appliqué à la finance}\\
Dépôt GitHub : \url{https://github.com/malaksabri65/24010191-SABRI-FIN3.git}
\end{center}

\begin{abstract}
Nous testons si les rendements et la volatilité passés du S\&P~500 permettent de prévoir le signe du rendement du lendemain mieux que deux règles naïves. Une régression logistique est évaluée sur un test chronologique utilisé une seule fois, avec un test binomial de significativité. Sur la période de test, l'accuracy est de \XX{xx,x}\,\% contre \XX{xx,x}\,\% pour la meilleure référence (écart de \XX{x,xx} points, $p=\XX{0,xxx}$). L'objectif fixé \XX{est atteint / n'est pas atteint}.
\end{abstract}

\tableofcontents
\newpage

%====================================================================
\section{Sujet et problématique}
\paragraph{Intérêt.} Si les rendements d'un indice boursier suivent une marche aléatoire, aucune règle fondée sur les cours passés ne peut battre le hasard (hypothèse d'efficience faible). Un gérant ou un étudiant en finance veut savoir si des informations simples (rendements et volatilité récents) permettent de prévoir le signe du lendemain, avant de bâtir une stratégie.

\paragraph{Problématique.} Un modèle de classification utilisant uniquement les rendements passés du S\&P~500 prévoit-il le signe du rendement du jour suivant mieux que la référence naïve, sur une période de test postérieure à l'apprentissage~?

\paragraph{Objectif mesurable (fixé avant de voir le test).} Métrique principale : accuracy sur le test (20\,\% des données les plus récentes). Réussite si l'accuracy dépasse d'au moins 2 points la meilleure des deux références (classe majoritaire de l'apprentissage ; signe de la veille) \emph{et} si un test binomial unilatéral contre cette référence est significatif à 5\,\%. Métriques secondaires : F1 et AUC.

\paragraph{Nature du problème.} Classification binaire sur série temporelle : la cible vaut 1 si le rendement du jour suivant est strictement positif, 0 sinon. Le découpage est chronologique. Algorithme principal : régression logistique (n$^\circ$~8) ; comparaison facultative : forêt aléatoire (n$^\circ$~12).

%====================================================================
\section{Jeu de données}
\begin{itemize}
  \item \textbf{Source} : Federal Reserve Bank of St.\ Louis (FRED), série \texttt{SP500}, cours de clôture quotidiens du S\&P~500. Lien : \url{https://fred.stlouisfed.org/series/SP500}.
  \item \textbf{Date d'extraction} : \XX{jj/mm/aaaa}. \textbf{Licence} : données S\&P Dow Jones Indices diffusées par FRED, usage pédagogique \XX{à confirmer sur la page Notes}.
  \item \textbf{Taille et fréquence} : \XX{N} observations quotidiennes, du \XX{date début} au \XX{date fin}. \textbf{Unité d'observation} : un jour de bourse.
  \item \textbf{Adéquation} : la série de clôtures suffit à construire la cible et les variables passées.
  \item \textbf{Limites} : environ dix ans seulement ; une seule série ; période incluant le krach de mars 2020 ; prix sans dividendes ; aucun coût de transaction.
\end{itemize}

\begin{table}[H]
\centering\small
\begin{tabular}{p{2.7cm}p{6.3cm}lll}
\toprule
Variable & Description & Type & Unité & Rôle \\
\midrule
\texttt{prix} & Cours de clôture & numérique & points & brute \\
\texttt{ret} & Rendement log $100\ln(P_t/P_{t-1})$ & numérique & \% & base \\
\texttt{ret\_lag1..5} & \texttt{ret} décalé de 1 à 5 jours & numérique & \% & explicative \\
\texttt{moy5, moy20} & Moyenne glissante de \texttt{ret} (5 / 20 j) & numérique & \% & explicative \\
\texttt{vol5, vol20} & Écart-type glissant de \texttt{ret} (5 / 20 j) & numérique & \% & explicative \\
\texttt{y} & 1 si $\text{ret}_{t+1}>0$, sinon 0 & binaire & -- & \textbf{cible} \\
\bottomrule
\end{tabular}
\caption{Dictionnaire des variables.}
\end{table}

%====================================================================
\section{Exploration}
\subsection{Qualité des données}
\begin{table}[H]
\centering\small
\begin{tabular}{p{2.6cm}p{3.6cm}p{2.2cm}p{5cm}}
\toprule
Problème & Détection & Décision & Justification \\
\midrule
Valeurs manquantes & \texttt{isna()} sur le prix (\XX{n} lignes) & Supprimées & Jours fériés sans cotation ; imputer créerait des rendements nuls artificiels \\
Doublons & \texttt{duplicated()} (\XX{0}) & Aucun & Une ligne par jour \\
Valeurs atypiques & Rendements à plus de $3\sigma$ (\XX{n} jours) & \textbf{Conservées} & Événements réels (krach, rebonds) ; les supprimer fausserait l'évaluation \\
\bottomrule
\end{tabular}
\caption{Contrôle qualité.}
\end{table}

\subsection{Graphiques}
\fig{fig_prix.png}{S\&P 500 : cours de clôture quotidien.}{0.85}
\textbf{Interprétation.} \XX{Recopier l'interprétation générée par le notebook : évolution du prix, plus forte baisse depuis un sommet, non-stationnarité du prix.}

\fig{fig_distribution.png}{Distribution des rendements logarithmiques quotidiens.}{0.85}
\textbf{Interprétation.} \XX{Moyenne, écart-type, kurtosis, jours extrêmes conservés car réels.}

\fig{fig_autocorrelation.png}{Autocorrélation des rendements (bornes à 95\,\%).}{0.85}
\textbf{Interprétation.} \XX{Nombre de retards significatifs sur 20 ; conclusion cohérente ou non avec la marche aléatoire ; part de jours de hausse.}

%====================================================================
\section{Préparation}
\begin{itemize}
  \item \textbf{Variables construites avec le passé seulement} : retards avec \texttt{shift(k)}, $k>0$, fenêtres glissantes se terminant en $t$.
  \item \textbf{Cible décalée vers le futur} (\texttt{shift(-1)}), puis suppression de la dernière ligne ; l'alignement est vérifié par un affichage (date, variables, cible) et une assertion.
  \item \textbf{Découpage chronologique} 60\,\%/20\,\%/20\,\% sans mélange, avec un écart d'une observation entre les blocs : apprentissage du \XX{début} au \XX{fin}, validation du \XX{début} au \XX{fin}, test du \XX{début} au \XX{fin}.
  \item \textbf{Standardisation} dans un \texttt{Pipeline}, donc ajustée sur l'apprentissage seulement, y compris dans chaque pli de validation croisée. Elle est nécessaire car la pénalisation de la régression logistique est sensible aux échelles.
  \item Aucun encodage : toutes les variables sont numériques.
\end{itemize}

%====================================================================
\section{Algorithme : régression logistique}
Le modèle estime la probabilité que le rendement de demain soit positif :
\begin{equation}
P(y=1\mid x)=\sigma(z)=\frac{1}{1+e^{-z}},\qquad z=\beta_0+\beta^{\top}x ,
\end{equation}
où $x\in\mathbb{R}^p$ regroupe les variables du jour, $\beta_0$ est la constante, $\beta$ les coefficients \emph{appris}, et $\sigma$ la sigmoïde qui ramène $z$ dans $[0,1]$. Les paramètres minimisent la perte logistique pénalisée :
\begin{equation}
\min_{\beta_0,\beta}\ \|\beta\|_q^q + C\sum_{i=1}^{n}\Big[-y_i\ln p_i-(1-y_i)\ln(1-p_i)\Big],\qquad p_i=\sigma(\beta_0+\beta^{\top}x_i),
\end{equation}
avec $q=2$ (pénalité L2) ou $q=1$ (pénalité L1), $n$ le nombre d'observations d'apprentissage et $p_i$ la probabilité prédite pour l'observation $i$. La prédiction est « hausse » si $p\ge s$, avec le seuil $s=0{,}5$.

\paragraph{Hyperparamètres.} (1) $C$, inverse de la régularisation : petit, il rétrécit les coefficients (modèle simple, risque de sous-apprentissage) ; grand, il libère les coefficients (risque de surapprentissage). (2) Le type de pénalité : L2 réduit tous les coefficients, L1 en annule certains (sélection de variables). Le seuil $s$ n'est pas appris.

\paragraph{Pourquoi cet algorithme.} Peu de signal et beaucoup de bruit : un modèle simple, peu flexible et interprétable limite le surapprentissage. \textbf{Alternative écartée} : la forêt aléatoire, plus flexible, s'ajuste facilement au bruit sur $\approx$2\,500 observations peu informatives ; elle n'est gardée que comme comparaison. KNN a été écarté car la notion de voisin est peu informative en dimension 10 avec du bruit.

%====================================================================
\section{Protocole expérimental}
\begin{itemize}
  \item \textbf{Références naïves} : (a) classe majoritaire de l'apprentissage ; (b) signe de la veille.
  \item \textbf{Réglage} : \texttt{GridSearchCV} avec \texttt{TimeSeriesSplit} (5 plis) sur apprentissage et validation ; $C\in\{10^{-3},10^{-2},10^{-1},1,10\}$ et pénalité $\in\{\text{L1},\text{L2}\}$ (10 configurations).
  \item \textbf{Métrique principale} : accuracy (part de jours bien classés, sans unité). \textbf{Secondaires} : F1 et AUC (0,5 = hasard).
  \item \textbf{Incertitude} : moyenne et écart-type sur les plis ; test binomial sur le test.
  \item Test utilisé \textbf{une seule fois}. \textbf{Nombre total de configurations essayées} (data snooping) : \XX{16, voir journal du notebook}.
\end{itemize}

%====================================================================
\section{Résultats}
\subsection{Réglage}
Meilleur réglage : \XX{C = ..., pénalité = ...} ; accuracy de validation croisée $=\XX{0,xxx}\pm\XX{0,xxx}$.
\fig{fig_validation.png}{Courbe de validation : accuracy selon $C$ et la pénalité.}{0.85}

\subsection{Tableau comparatif}
\begin{table}[H]
\centering
\begin{tabular}{lrrrrrr}
\toprule
 & \multicolumn{3}{c}{Validation} & \multicolumn{3}{c}{Test} \\
\cmidrule(lr){2-4}\cmidrule(lr){5-7}
Modèle & Acc. & F1 & AUC & Acc. & F1 & AUC \\
\midrule
Réf.~a : classe majoritaire & \XX{} & \XX{} & -- & \XX{} & \XX{} & -- \\
Réf.~b : signe de la veille & \XX{} & \XX{} & -- & \XX{} & \XX{} & -- \\
Régression logistique       & \XX{} & \XX{} & \XX{} & \XX{} & \XX{} & \XX{} \\
\bottomrule
\end{tabular}
\caption{Références et modèle, en validation et en test.}
\end{table}
Écart à la meilleure référence : \XX{+x,xx} points (seuil : $+2$). Test binomial : $p=\XX{0,xxx}$ face à la référence, $p=\XX{0,xxx}$ face au hasard (50\,\%). \textbf{Interprétation} : \XX{reprendre le commentaire du notebook}.

\subsection{Analyse des erreurs}
\fig{fig_confusion.png}{Matrice de confusion et probabilités prédites (test).}{0.9}
\textbf{Interprétation.} \XX{Part de jours prédits en hausse, comportement proche de la référence majoritaire ou non, dispersion des probabilités.}

\fig{fig_erreurs_periodes.png}{Accuracy du test par année et selon le régime de volatilité.}{0.9}
\textbf{Interprétation.} \XX{Variations par année et par régime, à interpréter avec prudence (peu de jours par groupe).}

\subsection{Généralisation et stabilité}
Accuracy à l'ajustement : \XX{xx,x}\,\% ; en validation croisée : \XX{xx,x}\,\% ; au test : \XX{xx,x}\,\%. Sur les deux moitiés du test : \XX{xx,x}\,\% puis \XX{xx,x}\,\%. \XX{Conclure sur le surapprentissage et la stabilité.}

\subsection{Variables et seuil de décision}
\fig{fig_variables.png}{Coefficients et importance par permutation (validation).}{0.95}
\textbf{Interprétation prudente.} \XX{Variable la plus influente, coefficients proches de 0 ou non.} Un coefficient mesure une association statistique, non une relation de cause à effet. Le seuil 0,5 est conservé : le modifier d'après le test serait du data snooping.

\subsection{Illustration sur cinq observations non vues}
\begin{table}[H]
\centering\small
\begin{tabular}{lrrlll}
\toprule
Date & ret (\%) & $P(\text{hausse})$ & Prévision & Réalité & Correct \\
\midrule
\XX{} & \XX{} & \XX{} & \XX{} & \XX{} & \XX{} \\
\XX{} & \XX{} & \XX{} & \XX{} & \XX{} & \XX{} \\
\XX{} & \XX{} & \XX{} & \XX{} & \XX{} & \XX{} \\
\XX{} & \XX{} & \XX{} & \XX{} & \XX{} & \XX{} \\
\XX{} & \XX{} & \XX{} & \XX{} & \XX{} & \XX{} \\
\bottomrule
\end{tabular}
\caption{Cinq prévisions sur des jours de la fin du test.}
\end{table}

%====================================================================
\section{Analyse critique}
\begin{itemize}
  \item \textbf{Données} : un seul indice, environ dix ans, un seul découpage de test ; période incluant le krach de 2020.
  \item \textbf{Métrique} : l'accuracy traite tous les jours de la même façon ; un modèle à 53\,\% peut perdre de l'argent si ses erreurs tombent sur les gros mouvements ; aucun coût de transaction.
  \item \textbf{Modèle} : linéaire, dix variables issues du seul passé du prix.
  \item \textbf{Data snooping} : $\XX{16}$ configurations ont été essayées ; plus il y en a, plus un résultat favorable peut être dû au hasard. La significativité est évaluée sur un test utilisé une seule fois et le seuil n'a pas été ajusté sur le test.
  \item \textbf{Pistes} : fenêtres glissantes sur plusieurs périodes de test ; variables exogènes (VIX, taux) ; stratégie de trading avec coûts et ratio de Sharpe.
\end{itemize}

%====================================================================
\section{Conclusion}
Sur le test (\XX{dates}, \XX{n} jours), la régression logistique prévoit le signe du lendemain avec \XX{xx,x}\,\% d'accuracy contre \XX{xx,x}\,\% pour la meilleure référence (\XX{+x,xx} points, $p=\XX{0,xxx}$). L'objectif ($\ge +2$ points et $p<0{,}05$) \XX{est / n'est pas} atteint. \XX{Ce résultat est cohérent / n'est pas cohérent avec l'hypothèse de marche aléatoire.} Face à une série proche d'une marche aléatoire, un résultat négatif honnête est le résultat attendu ; un score légèrement supérieur à 50\,\% ne prouve rien sans test de significativité ni confirmation sur d'autres périodes.

%====================================================================
\section{Références et reproduction}
\begin{itemize}
  \item Federal Reserve Bank of St.\ Louis, FRED, série SP500, \url{https://fred.stlouisfed.org/series/SP500}.
  \item Support de cours \texttt{Tp\_ML.ipynb} et « Principaux algorithmes de Machine Learning » (2026).
  \item Documentation scikit-learn (\texttt{LogisticRegression}, \texttt{TimeSeriesSplit}, \texttt{GridSearchCV}, \texttt{Pipeline}, \texttt{permutation\_importance}) et SciPy (\texttt{binomtest}).
  \item \textbf{Reproduction} : \texttt{pip install -r requirements.txt}, puis « Redémarrer et tout exécuter » dans \texttt{notebooks/projet\_F08.ipynb} ; graine \texttt{GRAINE = 42}.
\end{itemize}

\section*{Déclaration des ressources et outils d'IA}
\begin{table}[H]
\centering\small
\begin{tabular}{p{3cm}p{2.2cm}p{4.2cm}p{4.3cm}}
\toprule
Outil & Date & Usage précis & Vérification effectuée \\
\midrule
Assistant d'IA (Claude, Anthropic) & 17/11/2026 & Structure et code du notebook, commentaires générés à partir des sorties, explication de la régression logistique & \XX{code lu, exécuté, sorties vérifiées, commentaires relus et reformulés} \\
Support de cours \texttt{Tp\_ML.ipynb} & 2026 & Rappel des outils scikit-learn & Adapté au sujet F8 \\
\bottomrule
\end{tabular}
\end{table}

\end{document}
\end{document}
