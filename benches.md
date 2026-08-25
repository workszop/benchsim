# RAGAS - szybka ocena reference-free
RAGAS został zaprojektowany jako framework do automatycznej, w dużej mierze reference-free ewaluacji RAG. Jego klasyczna wersja skupia się na kilku wymiarach, m.in. jakości kontekstu, faithfulness i answer relevance, bez konieczności ręcznego oznaczania dużego zbioru odpowiedzi.

Idea jest prosta: zamiast pytać „czy odpowiedź jest identyczna jak gold answer?”, framework używa modelu do rozłożenia problemu na łatwiejsze pytania. Dla faithfulness sprawdza, czy twierdzenia z odpowiedzi są wspierane przez retrieved context. Dla answer relevance ocenia, czy odpowiedź rzeczywiście odpowiada na intencję pytania. Dzięki temu dobrze nadaje się do szybkiego porównywania konfiguracji, gdy nie mamy pełnego gold datasetu.

Kiedy używać: szybkie eksperymenty, baseline ewaluacyjny, monitoring próbek i sytuacje bez gotowego zbioru referencyjnego.

Główne ograniczenie: wynik jest zależny od modelu i promptów użytych jako evaluator. W 2025 r. duże badania nad LLM-as-a-Judge pokazały znaczną zmienność między modelami i zadaniami, dlatego judge powinien być walidowany na ludzkich ocenach.

# ARES - wykorzystanie oceny ludzkiej w procesie półautomatycznym
ARES jest modelowym przykładem ewaluacji hybrydowej. Wymaga: zbioru dokuementów z docelowej domeny, kilku przykładów pytań/odpowiedzi do promptowania generatora danych oraz zbiór preferencji użytkownika - w pracy autorzy wskazują około 150 lub więcej oznaczonych przykładów. [Artykuł źródłowy] 

Pipeline ARES składa się z trzech etapów:

Generacja danych syntetycznych. Z domenowych dokumentów generowane są pytania i odpowiedzi. Powstają przykłady pozytywne oraz negatywne.

Trening trzech osobnych judge’ów. W publikacji użyto DeBERTa-v3-Large z binarnym classification head. Każdy model ocenia inny wymiar: Context Relevance, Answer Faithfulness i Answer Relevance.

Masowa ocena outputów RAG i korekta przez Prediction-Powered Inference. Judge oznacza dużą pulę nieopisanych danych, a mały zbiór human labels służy do oszacowania jego błędu. PPI wykorzystuje oba źródła do stworzenia skorygowanego oszacowania oraz przedziałów ufności.

W publikacji autorzy używają 95% przedziałów ufności, a do rankingu systemów biorą środkową wartość z przedziału. Eksperyment pokazał, że przy j mniej niż 100 - 150 etykietach ARES traci zdolność do sensownego rozróżniania podobnych konfiguracji. W wielu głównych eksperymentach używano 300 etykiet.

# IFEval - automatyczna ocena wykonywania instrukcji
IFEval służy do sprawdzania, czy model wykonuje jawne i jednoznacznie weryfikowalne instrukcje zawarte w prompcie. Zamiast korzystać z oceny człowieka albo modelu typu LLM-as-a-Judge, benchmark stosuje deterministyczne reguły, które zwracają wynik binarny: warunek został spełniony albo nie. [Artykuł źródłowy](https://arxiv.org/abs/2311.07911)

Oryginalny zbiór obejmuje 541 promptów zbudowanych z 25 typów instrukcji. Są to m.in. ograniczenia długości, wymagana liczba akapitów lub punktów, użycie wskazanych słów, zakaz stosowania określonych wyrazów albo znaków interpunkcyjnych, zapis całej odpowiedzi małymi literami czy zwrócenie wyniku w formacie JSON. Jeden prompt może zawierać kilka warunków, dzięki czemu benchmark sprawdza również, czy model potrafi spełnić je jednocześnie.

Ocena jest liczona na dwóch poziomach. Instruction-level accuracy określa odsetek pojedynczych warunków wykonanych poprawnie. Prompt-level accuracy uznaje odpowiedź za poprawną tylko wtedy, gdy spełnia ona wszystkie instrukcje w danym prompcie. Druga miara jest bardziej wymagająca i lepiej pokazuje, czy model nie pomija części polecenia przy bardziej złożonych zadaniach.

IFEval raportuje wariant strict i loose. Wariant strict sprawdza surową odpowiedź. Wariant loose wykonuje dodatkowo proste transformacje, np. usuwa znaczniki formatowania Markdown, pierwszą linię z wprowadzeniem typu „Oczywiście, oto odpowiedź” albo ostatnią linię z zakończeniem. Ogranicza to liczbę wyników fałszywie negatywnych, ale może zwiększać liczbę wyników fałszywie pozytywnych, dlatego oba wyniki powinny być analizowane razem.

Kiedy używać: porównywanie modeli i promptów pod kątem precyzyjnego wykonywania poleceń, testowanie zgodności z wymaganym formatem oraz regresyjne sprawdzanie, czy nowa konfiguracja nie pogorszyła przestrzegania ograniczeń.

Główne ograniczenie: IFEval mierzy zgodność z warunkami, a nie jakość merytoryczną odpowiedzi. Model może uzyskać punkt za poprawny format, mimo że podał błędną albo mało użyteczną treść. Benchmark obejmuje też głównie łatwe do zaprogramowania ograniczenia powierzchniowe, więc nie zastępuje oceny bardziej złożonych instrukcji, takich jak ton, intencja, poprawność rozumowania czy adekwatność odpowiedzi.

# QuestEval - wykorzystany w PLainBench 
W PLainBench wykorzystywane do badania zachowania kluczowych informacji po uproszczeniu tekstu, jest to ocena przez generowanie pytań i odpowiedzi.

QuestEval powstał jako metryka do oceny streszczeń, ale dobrze przenosi się na każde zadanie, w którym jeden tekst ma oddawać treść drugiego.

Idea jest prosta: jeśli tekst B wiernie oddaje treść tekstu A, to pytania faktograficzne ułożone do A powinny dać się odpowiedzieć na podstawie B. Zamiast pytać „czy te dwa teksty są podobne?”, framework generuje pytania, a następnie próbuje na nie odpowiedzieć, patrząc wyłącznie na tekst docelowy. 

Metryka jest dwukierunkowa. Kierunek w przód - pytania z tekstu źródłowego, odpowiedzi z wygenerowanego - mierzy kompletność: spadek oznacza, że model pominął treść. Kierunek wsteczny - pytania z tekstu wygenerowanego, odpowiedzi ze źródła - mierzy faithfulness: spadek oznacza twierdzenia niepokryte źródłem, czyli halucynacje. Wyniki można wykorzystywać oddzielnie lub połączyć np. średnią harmoniczną, żeby jedna wysoka nota nie zrekompensowała drugiej niskiej.

W przypadku PLainBencha nie zależy nam na pełnym pokryciu faktów, a przybliżone określenie, czy jakieś fakty nie giną, więc model proszony jest o wygenerowanie pięciu pytań i wzorcowej odpowiedzi na podstawie tekstu oryginalnego i pięciu pytań wraz z odpowiedzią na podstawie tekstu uproszczonego.

Dopasowanie odpowiedzi do gold answer liczy się jako pokrycie frazy z gold_answer - dla języków fleksyjnych konieczna jest przy tym lematyzacja.
