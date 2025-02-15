KMCD - Properties
=================

```k
requires "kmcd.md"

module KMCD-PROPS
    imports KMCD

    configuration
        <kmcd-properties>
          <kmcd/>
          <processed-events> .List </processed-events>
          <properties> #violationFSMs </properties>
        </kmcd-properties>
```

Measurables
-----------

### Lookup Defaulting to 0

Sometimes you need a lookup to default to zero, and want to cast the result as a `Wad`, `Ray`, or `Rad`.

```k
    syntax Wad ::= #lookupWad ( Map , Address ) [function]
 // ------------------------------------------------------
    rule #lookupWad( M,  A) => { M[A] }:>Wad requires A in_keys(M)
    rule #lookupWad(_M, _A) => wad(0)        [owise]

    syntax Ray ::= #lookupRay ( Map , Address ) [function]
 // ------------------------------------------------------
    rule #lookupRay( M,  A) => { M[A] }:>Ray requires A in_keys(M)
    rule #lookupRay(_M, _A) => ray(0)        [owise]

    syntax Rad ::= #lookupRad ( Map , Address ) [function]
 // ------------------------------------------------------
    rule #lookupRad( M,  A) => { M[A] }:>Rad requires A in_keys(M)
    rule #lookupRad(_M, _A) => rad(0)        [owise]
```

### Measure Event

```k
    syntax Event ::= Measure
    syntax Measure ::= Measure () [function]
                     | Measure ( debt: Rad , vice: Rad ,controlDai: Map , potChi: Ray , potPie: Wad , sumOfScaledArts: Rad ,  endDebt: Rad
                               , sumOfAllFlapLots: Rad , dai: Map , sumOfAllFlapBids: Wad , mkrBalances: Map , ash: Rad, sumOfDais: Rad )[klabel(LogMeasure), symbol]
 // -------------------------------------------------------------------------------------------------------------------------------------------------
    rule [[ Measure() => Measure(... debt: DEBT, vice: VAT_VICE, controlDai: controlDais(keys_list(VAT_DAIS)), potChi: POT_CHI, potPie: POT_PIE
                                , sumOfScaledArts: calcSumOfScaledArts(VAT_ILKS, VAT_URNS),  endDebt: END_DEBT
                                , sumOfAllFlapLots: sumOfAllFlapLots(FLAP_BIDS), dai: VAT_DAIS, sumOfAllFlapBids: sumOfAllFlapBids(FLAP_BIDS)
                                , mkrBalances: mkrBalances(), ash: VOW_ASH, sumOfDais: calcSumOfDais(VAT_DAIS)) ]]
         <vat-debt>  DEBT      </vat-debt>
         <vat-dai>   VAT_DAIS  </vat-dai>
         <vat-ilks>  VAT_ILKS  </vat-ilks>
         <vat-urns>  VAT_URNS  </vat-urns>
         <vat-vice>  VAT_VICE  </vat-vice>
         <pot-chi>   POT_CHI   </pot-chi>
         <pot-pie>   POT_PIE   </pot-pie>
         <end-debt>  END_DEBT  </end-debt>
         <flap-bids> FLAP_BIDS </flap-bids>
         <vow-ash>   VOW_ASH   </vow-ash>
```

### Dai in Circulation

State predicates that capture undesirable states in the system (representing violations of certain invariants).

```k
    syntax Map ::= controlDais    ( List       ) [function]
                 | controlDaisAux ( List , Map ) [function]
 // -------------------------------------------------------
    rule controlDais(USERS) => controlDaisAux(USERS, .Map)

    rule controlDaisAux(.List               , USER_DAIS) => USER_DAIS
    rule controlDaisAux(ListItem(ADDR) REST , USER_DAIS) => controlDaisAux(REST, USER_DAIS [ ADDR <- controlDaiForUser(ADDR) ])

    syntax Rad ::= controlDaiForUser ( Address ) [function]
                 | vatDaiForUser     ( Address ) [function]
                 | erc20DaiForUser   ( Address ) [function]
                 | potDaiForUser     ( Address ) [function]
 // -------------------------------------------------------
    rule controlDaiForUser(ADDR) => (vatDaiForUser(ADDR) +Rad potDaiForUser(ADDR)) +Rad erc20DaiForUser(ADDR)

    rule    vatDaiForUser(_)    => rad(0) [owise]
    rule [[ vatDaiForUser(ADDR) => VAT_DAI ]]
         <vat-dai> ... ADDR |-> VAT_DAI:Rad ... </vat-dai>

    rule potDaiForUser(ADDR) => vatDaiForUser(Pot) *Rad Wad2Rad(portionOfPie(ADDR))

    rule    erc20DaiForUser(_)    => rad(0) [owise]
    rule [[ erc20DaiForUser(ADDR) => Wad2Rad(USER_ADAPT_DAI) ]]
         <dai-balance> ... ADDR |-> USER_ADAPT_DAI:Wad ... </dai-balance>

    syntax Wad ::= portionOfPie ( Address ) [function]
 // --------------------------------------------------
    rule    portionOfPie(_)    => wad(0) [owise]
    rule [[ portionOfPie(ADDR) => USER_PIE /Wad PIE ]]
         <pot-pies> ... ADDR |-> USER_PIE:Wad ... </pot-pies>
         <pot-pie> PIE </pot-pie>
      requires PIE =/=Wad wad(0)
       andBool ADDR =/=K Pot
```

### MKR Balances

By default, we assume the MKR balances are negative, but otherwise just grab the `<gem-balances>` cell for MKR.

```k
    syntax Map ::= mkrBalances() [function]
 // ---------------------------------------
    rule    mkrBalances() => .Map            [owise]
    rule [[ mkrBalances() => MKR_BALANCES ]]
         <gem>
           <gem-id> "MKR" </gem-id>
           <gem-balances> MKR_BALANCES </gem-balances>
           ...
         </gem>
```

### Vat Measures

Art of an ilk = Sum of all urn art across all users for that ilk.

```k
    syntax Wad ::= sumOfUrnArt(Map, String, Wad) [function, functional]
 // -------------------------------------------------------------------
    rule sumOfUrnArt( {ILK_ID , _:Address } |-> Urn (... art: ART) URNS, ILK_ID', SUM)
      => #if ILK_ID ==K ILK_ID'
            #then sumOfUrnArt( URNS, ILK_ID', SUM +Wad ART)
            #else sumOfUrnArt( URNS, ILK_ID', SUM)
         #fi

    rule sumOfUrnArt( _ |-> _ URNS, ILK_ID', SUM ) => sumOfUrnArt( URNS, ILK_ID', SUM ) [owise]

    rule sumOfUrnArt(.Map, _, SUM) => SUM
```

Total backed debt (sum over each CDP's art times corresponding ilk's rate)

```k
    syntax Rad ::= calcSumOfScaledArts   (      Map, Map     ) [function]
                 | calcSumOfScaledArtsAux(List, Map, Map, Rad) [function]
 // ---------------------------------------------------------------------
    rule calcSumOfScaledArts(VAT_ILKS, VAT_URNS) => calcSumOfScaledArtsAux(keys_list(VAT_ILKS), VAT_ILKS, VAT_URNS, rad(0))

    rule calcSumOfScaledArtsAux(                        .List ,        _ ,        _ , TOTAL ) => TOTAL
    rule calcSumOfScaledArtsAux( ListItem(ILK_ID) VAT_ILK_IDS , VAT_ILKS , VAT_URNS , TOTAL ) => calcSumOfScaledArtsAux(VAT_ILK_IDS, VAT_ILKS, VAT_URNS, TOTAL +Rad (sumOfUrnArt(VAT_URNS, ILK_ID, wad(0)) *Rate rate({VAT_ILKS[ILK_ID]}:>VatIlk)))
```

Total ammount of Dai issued: Sum of `VAT_DAIS`

```k
    syntax Rad ::= calcSumOfDais(Map) [function]
                 | calcSumOfDaisAux(List, Rad) [function]
 // -----------------------------------------------------------
    rule calcSumOfDais(VAT_DAIS) => calcSumOfDaisAux(values(VAT_DAIS), rad(0))

    rule calcSumOfDaisAux( .List, TOTAL) => TOTAL

    rule calcSumOfDaisAux( ListItem(AMOUNT) REST, SUM) => calcSumOfDaisAux(REST, SUM +Rad AMOUNT)

```

### Flap Measures

Sum of all lot values (i.e. total surplus dai up for auction).

```k
    syntax Rad ::= sumOfAllFlapLots   (      Map     ) [function]
                 | sumOfAllFlapLotsAux(List, Map, Rad) [function]
 // -------------------------------------------------------------
    rule sumOfAllFlapLots(FLAP_BIDS) => sumOfAllFlapLotsAux(keys_list(FLAP_BIDS), FLAP_BIDS, rad(0))

    rule sumOfAllFlapLotsAux(                          .List ,         _ , SUM ) => SUM
    rule sumOfAllFlapLotsAux( ListItem(BID_ID) FLAP_BIDS_IDS , FLAP_BIDS , SUM ) => sumOfAllFlapLotsAux(FLAP_BIDS_IDS, FLAP_BIDS, SUM +Rad lot({FLAP_BIDS[BID_ID]}:>FlapBid))
```

Sum of all bid values (i.e. total amount of MKR that's been bid on dai currently up for auction).

```k
    syntax Wad ::= sumOfAllFlapBids   (      Map     ) [function]
                 | sumOfAllFlapBidsAux(List, Map, Wad) [function]
 // -------------------------------------------------------------
    rule sumOfAllFlapBids(FLAP_BIDS) => sumOfAllFlapBidsAux(keys_list(FLAP_BIDS), FLAP_BIDS, wad(0))

    rule sumOfAllFlapBidsAux(                          .List ,         _ , SUM ) => SUM
    rule sumOfAllFlapBidsAux( ListItem(BID_ID) FLAP_BIDS_IDS , FLAP_BIDS , SUM ) => sumOfAllFlapBidsAux(FLAP_BIDS_IDS, FLAP_BIDS, SUM +Wad bid({FLAP_BIDS[BID_ID]}:>FlapBid))
```

Violations
----------

A violation occurs if any of the properties above holds.

```k
    syntax Map ::= "#violationFSMs" [function]
 // ------------------------------------------
    rule #violationFSMs => ( "Zero-Time Pot Interest Accumulation" |-> zeroTimePotInterest                           )
                           ( "Pot Interest Accumulation After End" |-> potEndInterest                                )
                           ( "Unauthorized Flip Kick"              |-> unAuthFlipKick                                )
                           ( "Unauthorized Flap Kick"              |-> unAuthFlapKick                                )
                           ( "Total Bound on Debt"                 |-> totalDebtBounded(... dsr: ray(1))             )
                           ( "PotChi PotPie VatPot"                |-> potChiPieDai(... offset: rad(0))              )
                           ( "Fundamental Dai Equation"            |-> fundamentalDaiEquation                        )
                           ( "Total Backed Debt Consistency"       |-> totalBackedDebtConsistency                    )
                           ( "Debt Constant After Thaw"            |-> debtConstantAfterThaw                         )
                           ( "Flap Dai Consistency"                |-> flapDaiConsistency                            )
                           ( "Flap MKR Consistency"                |-> flapMkrConsistency                            )
                           ( "Flop Block Check"                    |-> flopBlockCheck(... embers: rad(0), dented: 0) )
```

A violation can be checked using the Admin step `assert`. If a violation is detected,
it is recorded in the state and execution is immediately terminated.

```k
    syntax AdminStep ::= "#assert" | "#assert-failure"
 // --------------------------------------------------
    rule <k> assert => deriveAll(keys_list(VFSMS), #extractAssertEvents(EVENTS ListItem(Measure()))) ~> #assert ... </k>
         <events> EVENTS => .List </events>
         <properties> VFSMS </properties>

    rule <k> #assert => . ... </k>
         <properties> VFSMS </properties>
      requires notBool anyViolation(values(VFSMS))

    rule <k> #assert => #assert-failure ... </k> [owise]

    syntax List ::= #extractAssertEvents ( List ) [function]
 // --------------------------------------------------------
    rule #extractAssertEvents(.List)                                                                               => .List
    rule #extractAssertEvents(ListItem(C:CustomEvent)                                                        REST) => ListItem(C)                                           #extractAssertEvents(REST)
    rule #extractAssertEvents(ListItem(M:Measure)                                                            REST) => ListItem(M)                                           #extractAssertEvents(REST)
    rule #extractAssertEvents(ListItem(TimeStep(_, _) #as T)                                                 REST) => ListItem(T)                                           #extractAssertEvents(REST)
    rule #extractAssertEvents(ListItem(Transaction(... acct: ADDR, call: C, events: ES, txException: false)) REST) => ListItem(LogNote(ADDR, C))   #extractAssertEvents(ES) #extractAssertEvents(REST)
    rule #extractAssertEvents(ListItem(Transaction(... acct: ADDR, call: C,             txException: true )) REST) => ListItem(Exception(ADDR, C))                          #extractAssertEvents(REST)
    rule #extractAssertEvents(ListItem(_)                                                                    REST) =>                                                       #extractAssertEvents(REST) [owise]
```

### Violation Finite State Machines (FSMs)

These Finite State Machines help track whether certain properties of the system are violated or not.
Every FSM is equipped one special state, `Violated`, which holds the prior state to being violated.

```k
    syntax ViolationFSM ::= Violated ( ViolationFSM )
 // -------------------------------------------------
```

You can inject `checkViolated(_)` steps to each FSM to see whether we should halt because that FSM has a violation.

```k
    syntax Bool ::= anyViolation ( List ) [function]
 // ------------------------------------------------
    rule anyViolation(.List)                   => false
    rule anyViolation(ListItem(Violated(_)) _) => true
    rule anyViolation(ListItem(_VFSM)    REST) => anyViolation(REST) [owise]
```

For each FSM, the user must define the `derive` function, which dictates how that FSM behaves.
A default `owise` rule is added which leaves the FSM state unchanged.

```k
    syntax ViolationFSM ::= derive ( ViolationFSM , Event ) [function]
 // ------------------------------------------------------------------
    rule derive(VFSM, _) => VFSM [owise]

    syntax AdminStep ::= deriveAll  ( List , List  )
                       | deriveVFSM ( List , Event )
 // ------------------------------------------------
    rule <k> deriveAll(_, .List) => . ... </k>
    rule <k> deriveAll(VFSMIDS, ListItem(E) REST)
          => deriveVFSM(VFSMIDS, E)
          ~> deriveAll(VFSMIDS, REST)
         ...
         </k>
         <processed-events> L => L ListItem(E) </processed-events>

    rule <k> deriveVFSM(.List                 , _) => .                   ... </k>
    rule <k> deriveVFSM(ListItem(VFSMID) REST , E) => deriveVFSM(REST, E) ... </k>
         <properties> ... VFSMID |-> (VFSM => derive(VFSM, E)) ... </properties>
```

### Fundamental Equation of Dai

Sum of Dai of all users should be equal to Vat.vice plus Sum of debts of all ilks of all users.

```k
    syntax ViolationFSM ::= "fundamentalDaiEquation"
 // ------------------------------------------------
    rule derive(fundamentalDaiEquation, Measure(... debt: DEBT, vice: VICE, sumOfDais: SUM))
    => Violated(fundamentalDaiEquation) requires SUM =/=Rad (DEBT +Rad VICE)
```

### Total Backed Debt Consistency

Vat.debt minus Vat.vice should equal the sum over all ilks and CDP accounts of the CDP's art times the ilk's rate.

```k
    syntax ViolationFSM ::= "totalBackedDebtConsistency"
 // ----------------------------------------------------
    rule derive(totalBackedDebtConsistency, Measure(... debt: DEBT,  vice: VICE, sumOfScaledArts: SUM)) => Violated(totalBackedDebtConsistency) requires SUM =/=Rad (DEBT -Rad VICE)
```

### Debt Constant After Thaw

Vat.debt should not change after End.thaw is called, as this implies the creation or destruction of dai which would mess up the End's accounting.

```k
    syntax ViolationFSM ::= "debtConstantAfterThaw"
 // -----------------------------------------------
    rule derive(debtConstantAfterThaw, Measure(... debt: DEBT, endDebt: END_DEBT)) => Violated(debtConstantAfterThaw) requires (END_DEBT =/=Rad rad(0)) andBool (DEBT =/=Rad END_DEBT)
```

### Bounded Debt Growth

The Debt growth should be bounded in principle by the interest rates available in the system.

```k
    syntax ViolationFSM ::= totalDebtBounded    (             dsr: Ray )
                          | totalDebtBoundedRun ( debt: Rad , dsr: Ray )
                          | totalDebtBoundedEnd ( debt: Rad            )
 // --------------------------------------------------------------------
    rule derive(totalDebtBounded(... dsr: DSR), Measure(... debt: DEBT)) => totalDebtBoundedRun(... debt: DEBT, dsr: DSR)

    rule derive( totalDebtBoundedRun(... debt: DEBT          ) #as PREV , Measure(... debt: DEBT')            ) => Violated(PREV) requires DEBT' >Rad DEBT
    rule derive( totalDebtBoundedRun(... debt: DEBT, dsr: DSR)          , TimeStep(TIME, _)                   ) => totalDebtBoundedRun(... debt: DEBT +Rad rmul(vatDaiForUser(Pot), (DSR ^Ray TIME) -Ray ray(1)), dsr: DSR)
    rule derive( totalDebtBoundedRun(...             dsr: DSR)          , LogNote(_ , Vat . frob _ _ _ _ _ _) ) => totalDebtBounded(... dsr: DSR)
    rule derive( totalDebtBoundedRun(... debt: DEBT, dsr: DSR)          , LogNote(_ , Vat . suck _ _ AMOUNT)  ) => totalDebtBoundedRun(... debt: DEBT +Rad AMOUNT, dsr: DSR)
    rule derive( totalDebtBoundedRun(... debt: DEBT          )          , LogNote(_ , Pot . file dsr DSR')    ) => totalDebtBoundedRun(... debt: DEBT, dsr: DSR')
    rule derive( totalDebtBoundedRun(... debt: DEBT          )          , LogNote(_ , End . cage         )    ) => totalDebtBoundedEnd(... debt: DEBT)

    rule derive(totalDebtBoundedEnd(... debt: DEBT) #as PREV, Measure(... debt: DEBT')) => Violated(PREV) requires DEBT' =/=Rad DEBT
```

### Pot Chi * Pot Pie == Vat Dai(Pot)

The Pot Chi multiplied by Pot Pie should equal the Vat Dai for the Pot (offset by any direct moves people make of Dai to the Pot).

```k
    syntax ViolationFSM ::= potChiPieDai ( offset: Rad )
 // ----------------------------------------------------
    rule derive(potChiPieDai(... offset: OFFSET), LogNote(_, Vat . move _ Pot AMOUNT)) => potChiPieDai(... offset: OFFSET +Rad AMOUNT)

    rule derive(potChiPieDai(... offset: OFFSET) #as PREV, Measure(... controlDai: CONTROL_DAI, potChi: POT_CHI, potPie: POT_PIE)) => Violated(PREV) requires POT_PIE *Rate POT_CHI =/=Rad #lookupRad(CONTROL_DAI, Pot) -Rad OFFSET
```

### Kicking off a fake `flip` auction (inspired by lucash-flip)

The property checks if `flip . kick` is ever called by an unauthorized user (alternatively, the property can check whether a `flip` auction is kicked off with a zero bid?).

```k
    syntax ViolationFSM ::= "unAuthFlipKick"
 // ----------------------------------------
    rule derive(unAuthFlipKick, FlipKick(ADDR, ILK, _, _, _, _, _, _)) => Violated(unAuthFlipKick) requires notBool isAuthorized(ADDR, Flip ILK)
```

### Kicking off a fake `flap` auction (inspired by lucash-flap)

The property checks if `flap . kick` is ever called by an unauthorized user (alternatively, the property can check whether a `flap` auction is kicked off with a zero bid?).

```k
    syntax ViolationFSM ::= "unAuthFlapKick"
 // ----------------------------------------
    rule derive(unAuthFlapKick, FlapKick(ADDR, _, _, _)) => Violated(unAuthFlapKick) requires notBool isAuthorized(ADDR, Flap)
```

### Earning interest from a pot after End is deactivated (inspired by the lucash-pot-end attack)

The property checks if an `End . cage` is eventually followed by a successful `Pot . file dsr`.

```k
    syntax ViolationFSM ::= "potEndInterest" | "potEndInterestEnd"
 // --------------------------------------------------------------
    rule derive(potEndInterest   , LogNote( _ , End . cage       )) => potEndInterestEnd
    rule derive(potEndInterestEnd, LogNote( _ , Pot . file dsr _ )) => Violated(potEndInterestEnd)
```

### Earning interest from a pot in zero time (inspired by the lucash-pot attack)

The property checks if a successful `Pot . join` is preceded by a `TimeStep` more recently than a `Pot . drip`.

```k
    syntax ViolationFSM ::= "zeroTimePotInterest" | "zeroTimePotInterestEnd"
 // ------------------------------------------------------------------------
    rule derive(zeroTimePotInterest, TimeStep(N,_)) => zeroTimePotInterestEnd
      requires N >Int 0

    rule derive(zeroTimePotInterestEnd, LogNote( _ , Pot . join _ )) => Violated(zeroTimePotInterestEnd)
    rule derive(zeroTimePotInterestEnd, LogNote( _ , Pot . drip   )) => zeroTimePotInterest
```

### Flap dai consistency

```k
    syntax ViolationFSM ::= "flapDaiConsistency" | "flapDaiConsistencyEnd"
 // ----------------------------------------------------------------------
    rule derive(flapDaiConsistency, Measure(... sumOfAllFlapLots: SUM, dai: VAT_DAI)) => Violated(flapDaiConsistency) requires (SUM >Rad #lookupRad(VAT_DAI, Flap))
    rule derive(flapDaiConsistency, LogNote(_ , End . cage)                         ) => flapDaiConsistencyEnd
```

### Flap MKR consistency

```k
    syntax ViolationFSM ::= "flapMkrConsistency"
 // --------------------------------------------
    rule derive(flapMkrConsistency, Measure(... sumOfAllFlapBids: SUM, mkrBalances: BALS)) => Violated(flapMkrConsistency) requires (SUM >Wad #lookupWad(BALS, Flap))
```

### Flop Blocking
```k
    syntax ViolationFSM ::= flopBlockCheck(embers: Rad, dented: Int)
 // ----------------------------------------------------------------
    rule derive(flopBlockCheck(... embers: EMBERS, dented: DENTED), Measure(... dai: VAT_DAI, ash: ASH)) => Violated(flopBlockCheck(... embers: EMBERS, dented: DENTED))
        requires (ASH -Rad #lookupRad(VAT_DAI, Vow) >Rad EMBERS)

    rule derive(flopBlockCheck(... embers: EMBERS, dented: DENTED), FlopKick(_, _, BID, _)) => flopBlockCheck(... embers: EMBERS +Rad BID, dented: DENTED)

    rule derive(flopBlockCheck(... embers: EMBERS, dented: DENTED), LogNote(_ , Flop . dent DENT_ID _ BID)) => flopBlockCheck(... embers: EMBERS -Rad BID, dented: DENTED +Int (2 ^Int DENT_ID))
        requires ( ( ( DENTED /Int ( 2 ^Int DENT_ID ) ) modInt 2 ) ==Int 0 )
    rule derive(flopBlockCheck(... embers: EMBERS, dented: DENTED), LogNote(_ , Flop . dent _ _ _)) => flopBlockCheck(... embers: EMBERS, dented: DENTED) [owise]
```

```k
endmodule
```
