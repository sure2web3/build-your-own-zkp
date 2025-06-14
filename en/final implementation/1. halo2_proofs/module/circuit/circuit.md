### Detailed Implementation of the Circuit Module
![alt text](circuit.png)

In Halo2, **columns** are fundamental structures that organize data in a circuit. There are three main types: **advice columns** (for private witness data provided by the prover), **instance columns** (for public inputs known to both the prover and verifier), and **fixed columns** (for precomputed constants or values set during circuit setup). A **cell** refers to a specific location within a column at a given row, identified by a column and a **rotation** (row offset). Cells are queried using `VirtualCells` to build constraints. A **region** is a contiguous block of rows where gates and constraints are applied. Columns and their cells are grouped into regions to define computational steps, with **selectors** enabling conditional enforcement of constraints within these regions. Together, these components form the backbone of circuit layout and constraint definition in Halo2.
#### 1. Core Types and Traits Definition
- **Column Types**: In `src/plonk/circuit.rs`, several column types are defined. `ColumnType` is a trait that represents a column type. There are three concrete column types: `Advice`, `Fixed`, and `Instance`, along with an enum `Any` that can represent any of these three types.
    ```rust
    pub trait ColumnType:
        'static + Sized + Copy + std::fmt::Debug + PartialEq + Eq + Into<Any>
    {
    }

    pub struct Advice;
    pub struct Fixed;
    pub struct Instance;

    pub enum Any {
        Advice,
        Fixed,
        Instance,
    }
    ```
    These types are used to distinguish different kinds of columns in the circuit, such as advice columns for witness values, fixed columns for pre - determined values, and instance columns for public inputs.

- **Column Structure**: The `Column` struct holds an index and a column type. It implements `Ord` and `PartialOrd` traits to ensure a deterministic ordering of columns, which is crucial for the consensus in the proving system.
    ```rust
    /// A column with an index and type
    pub struct Column<C: ColumnType> {
        index: usize,
        column_type: C,
    }
    ```

    ```mermaid
    classDiagram
        class ColumnType {
            <<trait>>
        }
        class Column {
            - index: usize
            - column_type: C
            + new(index: usize, column_type: C) Column< C>
            + index() usize
            + column_type() &C
            + cmp(other: &Column< C>) std::cmp::Ordering
            + partial_cmp(other: &Column< C>) Option< std::cmp::Ordering>
        }
        ColumnType <|.. Column : implements
    ```
    - **Attributes**:
        - `index: usize`: Represents the index of the column.
        - `column_type: C`: Represents the type of the column, where `C` is a type that implements the `ColumnType` trait.

    - **Methods**:
        - `new(index: usize, column_type: C) -> Column<C>`: A constructor method (only available in test configurations) that creates a new `Column` instance with the given index and column type.
        - `index() -> usize`: Returns the index of the column.
        - `column_type() -> &C`: Returns a reference to the column type.
        - `cmp(other: &Column<C>) -> std::cmp::Ordering`: Implements the `Ord` trait, which is used to compare two `Column` instances. The comparison first compares the column types, and if they are equal, it compares the indices.
        - `partial_cmp(other: &Column<C>) -> Option<std::cmp::Ordering>`: Implements the `PartialOrd` trait, which provides a partial comparison of two `Column` instances. It simply calls the `cmp` method.

- **Flowchart**:
    ```mermaid
    classDiagram
        %% Enum and marker types
        class Any {
            <<enum>>
            +Advice
            +Fixed
            +Instance
        }
        class Advice
        class Fixed
        class Instance

        %% Trait
        class ColumnType {
            <<trait>>
        }

        %% Trait implementations
        Advice ..|> ColumnType : Impl
        Fixed ..|> ColumnType : Impl
        Instance ..|> ColumnType : Impl
        Any ..|> ColumnType : Impl

        %% From conversions (type to enum)
        Advice ..> Any : From
        Fixed ..> Any : From
        Instance ..> Any : From

        %% Ordering
        Any ..|> Ord : Impl
        Any ..|> PartialOrd : Impl
    ```
    * `Advice`, `Fixed`, `Instance`, and `Any` all implement the `ColumnType` trait, which is used as a marker for valid column types.
    * This allows these types to be used wherever a generic parameter is required to implement ColumnType.
    * `Any` is an enum with three variants:     `Advice`, `Fixed`, and `Instance`. It is used to represent any column type generically.
    * `Advice`, `Fixed`, and `Instance` can be converted into `Any` using the From trait.
    * The `Any` enum implements `Ord` and `PartialOrd`, allowing columns to be sorted deterministically (with the order: `Instance` < `Advice` < `Fixed`). This is important for circuit layout determinism.

    ```mermaid
    classDiagram

        class Column_Advice
        class Column_Fixed
        class Column_Instance
        class Column_Any
        
        %% From conversions (Column<T> to Column<Any>)
        Column_Advice --|> Column_Any : From
        Column_Fixed --|> Column_Any : From
        Column_Instance --|> Column_Any : From

        %% TryFrom conversions (Column<Any> to Column<T>)
        Column_Any ..> Column_Advice : TryFrom
        Column_Any ..> Column_Fixed : TryFrom
        Column_Any ..> Column_Instance : TryFrom
    ```
    * `Column` is a generic struct parameterized by a `ColumnType` C (which can be Advice, Fixed, Instance, or Any).
    * There are conversions from `Column<Advice>`, `Column<Fixed>`, and `Column<Instance>` to `Column<Any>` via the `From` trait.
    * Conversely, you can attempt to convert a `Column<Any>` back to a specific column type using the `TryFrom` trait, which will fail if the type does not match.
    * ps: `Column_Fixed` means `Column<Fixed>`

#### 2. Selectors
- **Selector Definition**: A `Selector` represents a fixed boolean value per row of the circuit. It can be used to conditionally enable (portions of) gates. Selectors are disabled by default and need to be explicitly enabled on each row when required.
    ```rust
    #[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
    pub struct Selector(pub(crate) usize, bool);

    impl Selector {
        /// Enable this selector at the given offset within the given region.
        pub fn enable<F: Field>(&self, region: &mut Region<F>, offset: usize) -> Result<(), Error> {
            region.enable_selector(|| "", self, offset)
        }

        /// Is this selector "simple"? Simple selectors can only be multiplied
        /// by expressions that contain no other simple selectors.
        pub fn is_simple(&self) -> bool {
            self.1
        }
    }
    ```
    * Example Usage:
        * Assume we have a circuit with two advice columns `A` and `B`, and a selector `S`. We'll represent rows in the circuit horizontally and columns vertically.

            ```plaintext
            Row |  A  |  B  |  S  | Constraint
            ----------------------------------
            0  |  1  |  -  |  0  | Not enforced
            1  |  -  |  1  |  1  | Enforced
            ```
        - The `A` and `B` columns represent advice columns where values can be assigned.
        - The `S` column represents the selector. A value of `0` means the selector is disabled, and a value of `1` means it is enabled.
        - The `Constraint` column indicates whether the constraint `A == B` is enforced on that row.
    * Summary: 
        * The `Selector` is a powerful tool in circuit design that allows for conditional enforcement of constraints. 
        * By enabling or disabling the selector on specific rows, we can control which parts of the circuit are active and which constraints are enforced.

#### 3. Query Structures
- **Query Types**: There are three query types defined: `FixedQuery`, `AdviceQuery`, and `InstanceQuery`. These are used to query values from fixed, advice, and instance columns at a certain relative location.
* **Role**: They allow the circuit designer to precisely define how values from different types of columns (fixed, advice, and instance) are accessed and used in the constraints of the circuit. By specifying the query index, column index, and rotation, the circuit can perform complex computations and verifications based on the values stored in these columns.
    ```rust
    /// Query of fixed column at a certain relative location
    #[derive(Copy, Clone, Debug)]
    pub struct FixedQuery {
        /// Query index
        pub(crate) index: usize,
        /// Column index
        pub(crate) column_index: usize,
        /// Rotation of this query, specifies the relative row (e.g., current, previous, next).
        pub(crate) rotation: Rotation,
    }

    /// Query of advice column at a certain relative location
    #[derive(Copy, Clone, Debug)]
    pub struct AdviceQuery {
        /// Query index
        pub(crate) index: usize,
        /// Column index
        pub(crate) column_index: usize,
        /// Rotation of this query, specifies the relative row (e.g., current, previous, next).
        pub(crate) rotation: Rotation,
    }

    /// Query of instance column at a certain relative location
    #[derive(Copy, Clone, Debug)]
    pub struct InstanceQuery {
        /// Query index
        pub(crate) index: usize,
        /// Column index
        pub(crate) column_index: usize,
        /// Rotation of this query, specifies the relative row (e.g., current, previous, next).
        pub(crate) rotation: Rotation,
    }

    ```
    * Role:
        * `FixedQuery`: used to represent a query of a fixed column at a certain relative location within the circuit.  
            * Fixed columns in a circuit typically contain values that are known and fixed throughout the proving process. For example, these could be constants or pre - computed values.
        * `AdviceQuery`: designed to query an advice column at a specific relative location. 
            * Advice columns are used to hold witness values, which are private inputs to the circuit. These values are typically unknown to the verifier but are used by the prover to construct a valid proof.
        * `InstanceQuery`: used to query an instance column at a particular relative location. 
            * Instance columns hold public inputs to the circuit, which are known to both the prover and the verifier. These values are used to verify the correctness of the proof.

#### 4. Lookup Table Column
- **TableColumn**: The `TableColumn` struct represents a fixed column of a lookup table. Lookup tables are an important component in zero - knowledge proof systems as they allow for efficient verification of membership and equality relationships within a set of pre - defined values.
* The `inner` field is of type `Column<Fixed>`, which means it represents a fixed column in the circuit. Fixed columns are used to store values that are known and fixed throughout the proving process. 
* `enable_equality`: This method enables equality on the `TableColumn`. By calling this method, the circuit designer can enforce that values in the table column satisfy certain equality constraints. This is crucial for ensuring the correctness and security of the lookup operations performed on the table column.
* Lookup table columns are always "encumbered" by the lookup arguments they are used in. This means that they cannot simultaneously be used as general fixed columns. Once a fixed column is designated as a `TableColumn` and used in a lookup argument, it is reserved for that specific lookup operation and cannot be used for other general fixed - value assignments.
    ```rust
    #[derive(Clone, Copy, Debug, Eq, PartialEq, Hash, Ord, PartialOrd)]
    pub struct TableColumn {
        /// The fixed column that this table column is stored in.
        ///
        /// # Security
        ///
        /// This inner column MUST NOT be exposed in the public API, or else chip developers
        /// can load lookup tables into their circuits without default-value-filling the
        /// columns, which can cause soundness bugs.
        inner: Column<Fixed>,
    }

    impl TableColumn {
        /// It is marked as `pub(crate)` to restrict its access to the crate level.
        pub(crate) fn inner(&self) -> Column<Fixed> {
            self.inner
        }

        /// Enable equality on this TableColumn.
        pub fn enable_equality<F: Field>(&self, meta: &mut ConstraintSystem<F>) {
            meta.enable_equality(self.inner)
        }
    }
    ```
* A lookup table can be loaded into a `TableColumn` via the `Layouter::assign_table` method. This method allows the circuit designer to populate the table column with the necessary values. 
* Currently, a single `TableColumn` can only contain one lookup table. However, these columns can be used in multiple lookup arguments through the `ConstraintSystem::lookup` method. This flexibility enables the circuit designer to reuse the same table column in different parts of the circuit for different verification purposes.
    * Example: Let's assume we are working on a circuit that verifies some arithmetic operations. We have a lookup table that stores pre - computed squares of numbers from 0 to 9. We can use this table in different parts of the circuit to check if a certain value is a square of a number within this range.
        ```rust
        use halo2_proofs::{
            circuit::{Layouter, SimpleFloorPlanner},
            pasta::Fp,
            plonk::{Circuit, ConstraintSystem, TableColumn},
            poly::Rotation,
        };

        #[derive(Clone)]
        struct MyCircuitConfig {
            table: TableColumn,
            advice: Column<Advice>,
        }

        struct MyCircuit;

        impl Circuit<Fp> for MyCircuit {
            type Config = MyCircuitConfig;
            type FloorPlanner = SimpleFloorPlanner;

            fn without_witnesses(&self) -> Self {
                Self
            }

            fn configure(meta: &mut ConstraintSystem<Fp>) -> Self::Config {
                let advice = meta.advice_column();
                let table = meta.lookup_table_column();

                // First lookup argument
                meta.lookup(|meta| {
                    let a = meta.query_advice(advice, Rotation::cur());
                    vec![(a, table)]
                });

                // Second lookup argument
                meta.lookup(|meta| {
                    let b = meta.query_advice(advice, Rotation::prev());
                    vec![(b, table)]
                });

                Self::Config {
                    table,
                    advice: advice.index(),
                }
            }

            fn synthesize(
                &self,
                config: Self::Config,
                mut layouter: impl Layouter<Fp>,
            ) -> Result<(), halo2_proofs::plonk::Error> {
                layouter.assign_table(
                    || "Square lookup table",
                    |mut table| {
                        for i in 0..10 {
                            let square = Fp::from(i * i);
                            table.assign_cell(
                                || format!("Square of {}", i),
                                config.table,
                                i,
                                || halo2_proofs::circuit::Value::known(square),
                            )?;
                        }
                        Ok(())
                    },
                )?;

                layouter.assign_region(
                    || "Region with lookup checks",
                    |mut region| {
                        // Assign a value that is a square
                        region.assign_advice(
                            || "Advice value (square)",
                            config.advice,
                            0,
                            || halo2_proofs::circuit::Value::known(Fp::from(4)),
                        )?;

                        // Assign another value that is a square
                        region.assign_advice(
                            || "Advice value (square)",
                            config.advice,
                            1,
                            || halo2_proofs::circuit::Value::known(Fp::from(9)),
                        )?;

                        Ok(())
                    },
                )?;

                Ok(())
            }
        }
        ```
    * Explanation:
        * Single Table in `TableColumn`: In the `synthesize` method, we assign a single lookup table to the `TableColumn` (`config.table`). This table stores the squares of numbers from 0 to 9.
        * Multiple Lookup Arguments: In the `configure` method, we define two lookup arguments using the same `TableColumn`. The first lookup argument checks the current value in the advice column against the table, and the second lookup argument checks the previous value in the advice column against the table.

    * This shows that although a `TableColumn` can hold only one lookup table, it can be used in multiple lookup arguments to perform different checks in the circuit. 

#### 5. Circuit Assignment
- **Assignment Trait**: The `Assignment` trait enables a `Circuit` to direct a backend to assign witnesses for a constraint system. It provides a set of methods for managing `regions` and `namespaces`, operating on various columns (`instance`, `advice`, and `fixed` columns), and handling constraints.   
    * Through these methods, Assignment allows the Circuit to precisely guide the backend in performing witness assignment for the constraint system, which is an essential part of building and verifying circuits. For example:
        * in `src/plonk/prover.rs`, some methods of `Assignment` are implemented to synthesize the circuit and obtain witness information.
        * in `src/dev/graph.rs`, `Assignment` is implemented to construct the dot graph of the circuit.

    ```mermaid
    classDiagram
        class Assignment {
            +enter_region(name_fn: N)
            +exit_region()
            +enable_selector(annotation: A, selector: &Selector, row: usize) Result<（）, Error>
            +query_instance(column: Column< Instance>, row: usize) Result< Value< F>, Error>
            +assign_advice(annotation: A, column: Column< Advice>, row: usize, to: V) Result<（）, Error>
            +assign_fixed(annotation: A, column: Column< Fixed>, row: usize, to: V) Result<（）, Error>
            +copy(left_column: Column< Any>, left_row: usize, right_column: Column< Any>, right_row: usize) Result<（）, Error>
            +fill_from_row(column: Column< Fixed>, row: usize, to: Value< Assigned< F>>) Result<（）, Error>
            +push_namespace(name_fn: N)
            +pop_namespace(gadget_name: Option< String>)
        }

        class Column {
            +index: usize
            +column_type: C
        }

        class Selector {
            +0: usize
            +1: bool
        }

        class Value {
            +inner: Option< Assigned< F>>
        }

        class Assigned {
            <<enum>>
            +Zero
            +Trivial
            +Rational
            // TODO(sure2web3) perfect the details
        }

        class Error {
            +message: String
        }

        Assignment --> Column : Uses
        Assignment --> Selector : Uses
        Assignment --> Value : Uses
        Assignment --> Assigned : Uses
        Assignment --> Error : Returns
    ```
    
    | Method Name | Description | Parameters | Return Type | Role in Halo2 |
    | --- | --- | --- | --- | --- |
    | `enter_region` | Creates a new region and enters into it.  Used for organizing assignments into logical regions. Panics if currently in a region (if `exit_region` was not called). Not intended for downstream consumption; use [`Layouter::assign_region`] instead. | `name_fn: N` where `NR: Into<String>` and `N: FnOnce() -> NR`, name_fn is a closure that returns a name for the region. | `()` | In Halo2, regions are used to organize the layout of a circuit. This method is crucial for starting a new logical section within the circuit layout process, allowing for modular and structured circuit design. |
    | `exit_region` | Exits the current region. Panics if not currently in a region (if `enter_region` was not called). Not intended for downstream consumption; use [`Layouter::assign_region`] instead. | None | `()` | Marks the end of a logical section in the circuit layout. It ensures proper scoping and organization of the circuit regions, which is important for the correct synthesis and evaluation of the circuit. |
    | `enable_selector` | Enables a selector at the given row. | `annotation: A`, `selector: &Selector`, `row: usize` where `A: FnOnce() -> AR` and `AR: Into<String>` | `Result<(), Error>` | Selectors are used to conditionally enable parts of gates in the circuit. Enabling a selector at a specific row allows the circuit to activate certain constraints only when needed, providing flexibility in circuit design. |
    | `query_instance` | Queries the cell of an instance column at a particular absolute row. Returns the cell's value, if known. | `column: Column<Instance>`, `row: usize` | `Result<Value<F>, Error>` | Instance columns hold public input values. Querying these columns allows the circuit to access and use the public data during the proof generation and verification process, which is essential for zero - knowledge proofs. |
    | `assign_advice` | Assigns an advice column value (witness). | `annotation: A`, `column: Column<Advice>`, `row: usize`, `to: V` where `V: FnOnce() -> Value<VR>`, `VR: Into<Assigned<F>>`, `A: FnOnce() -> AR`, and `AR: Into<String>`, to is a closure that provides the value. | `Result<(), Error>` | Advice columns are used to store witness values, which are the private inputs in a zero - knowledge proof. Assigning values to these columns is a fundamental step in constructing the proof. |
    | `assign_fixed` | Assigns a value to a fixed column at a specific row. | `annotation: A`, `column: Column<Fixed>`, `row: usize`, `to: V` where `V: FnOnce() -> Value<VR>`, `VR: Into<Assigned<F>>`, `A: FnOnce() -> AR`, and `AR: Into<String>` | `Result<(), Error>` | Fixed columns have pre - determined values throughout the circuit. Assigning fixed values is important for setting up the constant parts of the circuit, such as coefficients in polynomial equations. |
    | `copy` | Assigns two cells to have the same value. | `left_column: Column<Any>`, `left_row: usize`, `right_column: Column<Any>`, `right_row: usize` | `Result<(), Error>` | This method is used to enforce equality between two cells in the circuit. It helps in creating relationships between different parts of the circuit, which is necessary for expressing complex constraints. |
    | `fill_from_row` | Fills a fixed `column` starting from the given `row` with value `to`. | `column: Column<Fixed>`, `row: usize`, `to: Value<Assigned<F>>` | `Result<(), Error>` | Useful for initializing or populating fixed columns with a specific value from a certain row onwards. This can simplify the process of setting up the constant parts of the circuit. |
    | `push_namespace` | Creates a new (sub)namespace and enters into it for organizing assignments. Not intended for downstream consumption; use [`Layouter::namespace`] instead. | `name_fn: N` where `NR: Into<String>` and `N: FnOnce() -> NR` | `()` | Namespaces are used to organize and isolate different parts of the circuit. Pushing a new namespace allows for better modularity and separation of concerns in the circuit design. |
    | `pop_namespace` | Exits out of the existing namespace. Not intended for downstream consumption; use [`Layouter::namespace`] instead. | `gadget_name: Option<String>` | `()` | Marks the end of a namespace, ensuring proper scoping and organization of the circuit components within namespaces. It helps in maintaining a clean and structured circuit layout. |

#### 6. FloorPlanner and V1
```mermaid
classDiagram
    %% Traits
    class FloorPlanner {
        <<trait>>
        +synthesize<F:Field, CS:Assignment<F>, C:Circuit<F>>(cs:&mut CS, circuit:&C, config:C::Config, constants:Vec<Column<Fixed>>):Result<(), Error>
    }
    
    class Layouter {
        <<trait>>
        - F: Field
        +assign_region<A, AR, N, NR>(name:N, assignment:A):Result<AR, Error>
        +assign_table<A, N, NR>(name:N, assignment:A):Result<(), Error>
        +constrain_instance(cell:Cell, instance:Column<Instance>, row:usize):Result<(), Error>
        +get_root():&mut Self::Root
        +push_namespace<NR, N>(name_fn:N)
        +pop_namespace(gadget_name:Option<String>)
    }
    
    class RegionLayouter {
        <<trait>>
        - F: Field
        +enable_selector<'v>(annotation:&'v (dyn Fn() -> String + 'v), selector:&Selector, offset:usize):Result<(), Error>
        +assign_advice<'v>(annotation:&'v (dyn Fn() -> String + 'v), column:Column<Advice>, offset:usize, to:&'v mut (dyn FnMut() -> Value<Assigned<F>> + 'v)):Result<Cell, Error>
        +assign_advice_from_constant<'v>(annotation:&'v (dyn Fn() -> String + 'v), column:Column<Advice>, offset:usize, constant:Assigned<F>):Result<Cell, Error>
        +assign_advice_from_instance<'v>(annotation:&'v (dyn Fn() -> String + 'v), instance:Column<Instance>, row:usize, advice:Column<Advice>, offset:usize):Result<(Cell, Value<F>), Error>
        +instance_value(instance:Column<Instance>, row:usize):Result<Value<F>, Error>
        +assign_fixed<'v>(annotation:&'v (dyn Fn() -> String + 'v), column:Column<Fixed>, offset:usize, to:&'v mut (dyn FnMut() -> Value<Assigned<F>> + 'v)):Result<Cell, Error>
        +constrain_constant(cell:Cell, constant:Assigned<F>):Result<(), Error>
        +constrain_equal(left:Cell, right:Cell):Result<(), Error>
    }
    
    class TableLayouter {
        <<trait>>
        - F: Field
        +assign_cell<'v>(annotation:&'v (dyn Fn() -> String + 'v), column:TableColumn, offset:usize, to:&'v mut (dyn FnMut() -> Value<Assigned<F>> + 'v)):Result<(), Error>
    }
    
    %% Main Classes
    class V1 {
        <<struct>>
        +synthesize<F:Field, CS:Assignment<F>, C:Circuit<F>>(cs:&mut CS, circuit:&C, config:C::Config, constants:Vec<Column<Fixed>>):Result<(), Error>
    }
    
    class V1Plan {
        <<struct>>
        - 'a
        - F: Field
        - CS: Assignment<F>
        - cs: &'a mut CS
        - regions: Vec<RegionStart>
        - constants: Vec<(Assigned<F>, Cell)>
        - table_columns: Vec<TableColumn>
        +new(cs:&'a mut CS):Result<Self, Error>
    }
    
    class SimpleFloorPlanner {
        <<struct>>
        +synthesize<F:Field, CS:Assignment<F>, C:Circuit<F>>(cs:&mut CS, circuit:&C, config:C::Config, constants:Vec<Column<Fixed>>):Result<(), Error>
    }
    
    class SingleChipLayouter {
        <<struct>>
        - 'a
        - F: Field
        - CS: Assignment<F>
        - cs: &'a mut CS
        - constants: Vec<Column<Fixed>>
        - regions: Vec<RegionStart>
        - columns: HashMap<RegionColumn, usize>
        - table_columns: Vec<TableColumn>
        - _marker: PhantomData<F>
        +new(cs:&'a mut CS, constants:Vec<Column<Fixed>>):Result<Self, Error>
    }
    
    class V1Pass {
        <<struct>>
        - 'p, 'a
        - F: Field
        - CS: Assignment<F>
        - pass: Pass<'p, 'a, F, CS>
        +measure(pass:&'p mut MeasurementPass):Self
        +assign(pass:&'p mut AssignmentPass<'p, 'a, F, CS>):Self
    }
    
    class MeasurementPass {
        <<struct>>
        - regions: Vec<RegionShape>
        +new():Self
        +assign_region<F:Field, A, AR>(assignment:A):Result<AR, Error>
    }
    
    class AssignmentPass {
        <<struct>>
        - 'p, 'a
        - F: Field
        - CS: Assignment<F>
        - plan: &'p mut V1Plan<'a, F, CS>
        - region_index: usize
        +new(plan:&'p mut V1Plan<'a, F, CS>):Self
        +assign_region<A, AR, N, NR>(name:N, assignment:A):Result<AR, Error>
        +assign_table<A, AR, N, NR>(name:N, assignment:A):Result<AR, Error>
        +constrain_instance(cell:Cell, instance:Column<Instance>, row:usize):Result<(), Error>
    }
    
    class SingleChipLayouterRegion {
        <<struct>>
        - 'r, 'a
        - F: Field
        - CS: Assignment<F>
        - layouter: &'r mut SingleChipLayouter<'a, F, CS>
        - region_index: RegionIndex
        - constants: Vec<(Assigned<F>, Cell)>
        +new(layouter:&'r mut SingleChipLayouter<'a, F, CS>, region_index:RegionIndex):Self
    }
    
    class V1Region {
        <<struct>>
        - 'r, 'a
        - F: Field
        - CS: Assignment<F>
        - plan: &'r mut V1Plan<'a, F, CS>
        - region_index: RegionIndex
        +new(plan:&'r mut V1Plan<'a, F, CS>, region_index:RegionIndex):Self
    }
    
    %% Helper Classes
    class RegionShape {
        <<struct>>
        - region_index: RegionIndex
        - columns: HashSet<RegionColumn>
        - row_count: usize
        +new(region_index:RegionIndex):Self
        +region_index():RegionIndex
        +columns():&HashSet<RegionColumn>
        +row_count():usize
    }
    
    class RegionColumn {
        <<enum>>
        - Column(Column<Any>)
        - Selector(Selector)
    }
    
    class Cell {
        <<struct>>
        - region_index: RegionIndex
        - row_offset: usize
        - column: Column<Any>
    }
    
    class Column {
        <<struct>>
        - index: usize
        - column_type: Any
    }
    
    class Any {
        <<enum>>
        - Advice
        - Instance
        - Fixed
    }
    
    class RegionIndex {
        <<struct>>
        - index: usize
    }
    
    class RegionStart {
        <<struct>>
        - start: usize
    }
    
    class Selector {
        <<struct>>
        - index: usize
    }
    
    class Value {
        <<struct>>
        - inner: Option<V>
        +unknown():Self
        +known(value:V):Self
    }
    
    class Assigned {
        <<struct>>
        - F: Field
        - numerator: F
        - denominator: F
        +evaluate():F
    }
    
    class Error {
        <<enum>>
        - NotEnoughColumnsForConstants
        - TableError(TableError)
        - Synthesis
        - ...
    }
    
    class TableColumn {
        <<struct>>
        - column: Column<Fixed>
        - rotation: Rotation
    }
    
    %% Allocation Classes
    class AllocatedRegion {
        <<struct>>
        - start: usize
        - length: usize
    }
    
    class EmptySpace {
        <<struct>>
        - start: usize
        - end: Option<usize>
        +range():Option<Range<usize>>
    }
    
    class Allocations {
        <<struct>>
        - allocations: BTreeSet<AllocatedRegion>
        +unbounded_interval_start():usize
        +free_intervals(start:usize, end:Option<usize>):Iterator<Item=EmptySpace>
    }
    
    class CircuitAllocations {
        <<type>>
        HashMap<RegionColumn, Allocations>
    }
    
    %% Relationships
    FloorPlanner <|.. V1 : implements
    FloorPlanner <|.. SimpleFloorPlanner : implements
    
    Layouter <|.. V1Pass : implements
    Layouter <|.. SingleChipLayouter : implements
    
    RegionLayouter <|.. RegionShape : implements
    RegionLayouter <|.. SingleChipLayouterRegion : implements
    RegionLayouter <|.. V1Region : implements
    
    TableLayouter <|.. SimpleTableLayouter : implements
    
    V1 o-- V1Plan : uses
    V1Plan o-- AssignmentPass : uses
    V1Plan o-- V1Region : uses
    V1Plan "1" *-- "n" TableColumn : contains
    V1Plan "1" *-- "n" Assigned : contains
    V1Plan "1" *-- "n" Cell : contains
    V1Plan "1" *-- "n" RegionStart : contains
    
    SingleChipLayouter o-- SingleChipLayouterRegion : uses
    SingleChipLayouter "1" *-- "n" TableColumn : contains
    SingleChipLayouter "1" *-- "n" RegionStart : contains
    SingleChipLayouter "1" *-- "n" Column : key(columns)
    SingleChipLayouter "1" *-- "n" RegionColumn : value(columns)
    
    V1Pass "1" -- "1" MeasurementPass : uses(measure)
    V1Pass "1" -- "1" AssignmentPass : uses(assign)
    
    MeasurementPass "1" *-- "n" RegionShape : contains
    
    AssignmentPass "1" -- "1" V1Plan : uses
    AssignmentPass "1" *-- "n" TableColumn : uses(assign_table)
    
    SingleChipLayouterRegion "1" -- "1" SingleChipLayouter : uses
    SingleChipLayouterRegion "1" *-- "n" Assigned : contains
    SingleChipLayouterRegion "1" *-- "n" Cell : contains
    
    V1Region "1" -- "1" V1Plan : uses
    V1Region "1" *-- "n" Assigned : contains
    V1Region "1" *-- "n" Cell : contains
    
    RegionShape "1" *-- "n" RegionColumn : contains
    RegionShape "1" -- "1" RegionIndex : contains
    
    Cell "1" -- "1" RegionIndex : contains
    Cell "1" -- "1" Column : contains
    
    Column "1" -- "1" Any : contains
    
    RegionColumn "1" -- "1" Column : variant(Column)
    RegionColumn "1" -- "1" Selector : variant(Selector)
    
    Allocations "1" *-- "n" AllocatedRegion : contains
    CircuitAllocations "1" *-- "1" RegionColumn : key
    CircuitAllocations "1" *-- "1" Allocations : value
    
    AllocatedRegion "1" -- "1" EmptySpace : relates
    EmptySpace "1" -- "1" Allocations : used by
    
    Selector "1" -- "1" RegionColumn : used by
    Selector "1" -- "1" RegionLayouter : used by(enable_selector)
    
    Value "1" -- "1" RegionLayouter : returns(assign_advice)
    Value "1" -- "1" Assigned : contained in
    
    Assigned "1" -- "1" Value : contained in
    Assigned "1" -- "1" RegionLayouter : used by(assign_advice_from_constant)
    
    Error "1" -- "1" FloorPlanner : returns(synthesize)
    Error "1" -- "1" Layouter : returns(assign_region)
    Error "1" -- "1" RegionLayouter : returns(enable_selector)
    
    TableColumn "1" -- "1" Column : contains
    TableColumn "1" -- "1" TableLayouter : used by(assign_cell)
```

- **FloorPlanner Trait**: The `FloorPlanner` trait defines a floor planning strategy for a circuit. It is chip - agnostic and applies its strategy to the circuit it is used within. The main function of this trait is the `synthesize` method, which takes a constraint system (`CS`), a circuit (`C`), a configuration object (`config`), and a list of fixed columns (`constants`). 
    * `synthesize` refers to the process of constructing a circuit by assigning values to the various components of the circuit and enforcing the necessary constraints. This process is crucial for creating zero - knowledge proofs, as it allows the prover to demonstrate knowledge of a solution to a given problem without revealing the actual solution.
    ```rust 
    pub trait FloorPlanner {
        /// Given the provided `cs`, synthesize the given circuit.
        ///
        /// `constants` is the list of fixed columns that the layouter may use to assign
        /// global constant values. These columns will all have been equality-enabled.
        fn synthesize<F: Field, CS: Assignment<F>, C: Circuit<F>>(
            /*
                cs: &mut CS: A mutable reference to an assignment backend (the constraint system).
                circuit: &C: A reference to the circuit to be synthesized.
                config: C::Config: The configuration object for the circuit.
                constants: Vec<Column<Fixed>>: A vector of fixed columns used for global constants.
            */
            cs: &mut CS,
            circuit: &C,
            config: C::Config,
            constants: Vec<Column<Fixed>>,
        ) -> Result<(), Error>;
    }
    ```
    * Internally, the `synthesize` method performs the following steps:
        - Instantiates a `Layouter` for the floor planner.
        - Performs any necessary setup or measurement tasks, which might involve one or more calls to `Circuit::default().synthesize(config, &mut layouter)`.
        - Calls `circuit.synthesize(config, &mut layouter)` exactly once.

    * In Halo2, the `FloorPlanner` is responsible for determining how the circuit components are arranged in the circuit layout. It helps in optimizing the use of resources such as columns and rows in the circuit, and it orchestrates the `synthesis` process of the circuit. Different implementations of the `FloorPlanner` can lead to different circuit layouts, which can affect the performance and efficiency of the circuit.

- **V1**: `V1` is the main, production-ready floor planner in Halo2. It uses a dual-pass approach: first, it measures the shape and requirements of each region (measurement pass), then it assigns regions and constants according to a greedy, first-fit strategy sorted by advice area (assignment pass). It optimizes layout by prioritizing regions with more advice columns. This ensures efficient and correct region placement and constant assignment.
    ```rust
    /// The version 1 [`FloorPlanner`] provided by `halo2`.
    ///
    /// - No column optimizations are performed. Circuit configuration is left entirely to the
    ///   circuit designer.
    /// - A dual-pass layouter is used to measures regions prior to assignment.
    /// - Regions are measured as rectangles, bounded on the cells they assign.
    /// - Regions are laid out using a greedy first-fit strategy, after sorting regions by
    ///   their "advice area" (number of advice columns * rows).
    #[derive(Debug)]
    pub struct V1;
    ```
    - **strategy**: This module provides algorithms and data structures for region placement within the V1 floor planner. It defines how regions are allocated in columns, tracks empty spaces, and implements the "slot in biggest advice first" strategy to efficiently lay out regions and minimize column contention. The strategy module is essential for the planning phase, ensuring that all regions are placed without overlap and that the circuit layout is as compact as possible.

- **V1Plan**: `V1Plan<'a, F, CS>` Stores state for the V1 floor planner, including the constraint system, region starts, constants to assign, and table columns. It coordinates the two-pass synthesis process.
    ```rust
    struct V1Plan<'a, F: Field, CS: Assignment<F> + 'a> {
        cs: &'a mut CS,
        /// Stores the starting row for each region.
        regions: Vec<RegionStart>,
        /// Stores the constants to be assigned, and the cells to which they are copied.
        constants: Vec<(Assigned<F>, Cell)>,
        /// Stores the table fixed columns.
        table_columns: Vec<TableColumn>,
    }
    ```

- **Pass**: The `Pass` enum represents the current phase of the V1 floor planner. It can be either a `Measurement` pass, which collects information about the shape and requirements of each region, or an `Assignment` pass, which performs the actual assignment of values to the circuit according to the plan. This abstraction allows the same interface to be used for both phases, switching behavior based on the current pass.
    ```rust
    #[derive(Debug)]
    enum Pass<'p, 'a, F: Field, CS: Assignment<F> + 'a> {
        Measurement(&'p mut MeasurementPass),
        Assignment(&'p mut AssignmentPass<'p, 'a, F, CS>),
    }
    ```

- **V1Pass**: V1Pass is a wrapper struct that encapsulates a single pass (either measurement or assignment) of the V1 layouter. It provides methods to construct a measurement or assignment pass and implements the Layouter trait, dispatching region and table assignments to the correct underlying logic depending on the phase.
    ```rust
    /// A single pass of the [`V1`] layouter.
    #[derive(Debug)]
    pub struct V1Pass<'p, 'a, F: Field, CS: Assignment<F> + 'a>(Pass<'p, 'a, F, CS>);
    ```
    - **Functions**:

    | Function Name         | Purpose/Role                                                                                  | Execution Logic                                                                                                      | Function in Halo2                                                                                      |
    |---------------------- |----------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
    | measure               | Constructs a `V1Pass` in measurement mode                                                    | Wraps a mutable reference to `MeasurementPass` in the `Pass::Measurement` variant                                   | Used to collect region shape information during the first pass                                         |
    | assign                | Constructs a `V1Pass` in assignment mode                                                     | Wraps a mutable reference to `AssignmentPass` in the `Pass::Assignment` variant                                     | Used to perform actual assignments during the second pass                                              |
    | assign_region         | Assigns a region in the circuit                                                              | Delegates to either `MeasurementPass::assign_region` or `AssignmentPass::assign_region` based on the current phase  | Handles region assignment, collecting shape info or performing assignments as needed                   |
    | assign_table          | Assigns a lookup table in the circuit                                                        | Delegates to `AssignmentPass::assign_table` in assignment phase; does nothing in measurement phase                  | Ensures tables are assigned and checked for consistency during circuit synthesis                       |
    | constrain_instance    | Constrains an advice cell to equal an instance cell                                          | Delegates to `AssignmentPass::constrain_instance` in assignment phase; does nothing in measurement phase            | Used to link advice cells to public inputs (instance columns)                                          |
    | get_root              | Returns a mutable reference to the root layouter                                             | Simply returns `self`                                                                                                | Required by the `Layouter` trait for recursive namespace management                                    |
    | push_namespace        | Pushes a namespace for debugging or logical grouping                                         | Calls `push_namespace` on the underlying constraint system in assignment phase; does nothing in measurement phase    | Helps organize and debug circuit synthesis by grouping related assignments                             |
    | pop_namespace         | Pops the current namespace                                                                   | Calls `pop_namespace` on the underlying constraint system in assignment phase; does nothing in measurement phase     | Complements `push_namespace` for managing logical/debugging scopes in the circuit                      |

- **MeasurementPass**: MeasurementPass is responsible for the first phase of the V1 floor planner. It collects the shape (columns and rows used) of each region by simulating the circuit layout without assigning actual values. This information is used to plan region placement and ensure efficient, conflict-free assignments in the next phase.
    ```rust
    /// Measures the circuit.
    #[derive(Debug)]
    pub struct MeasurementPass {
        regions: Vec<RegionShape>,
    }
    ```
    - **Functions**:

    | Function Name   | Purpose/Role                                                      | Execution Logic                                                                                                  | Function in Halo2                                                      |
    |-----------------|-------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
    | new             | Creates a new `MeasurementPass`                                   | Initializes the `regions` vector as empty and returns a new `MeasurementPass` instance                          | Used to start the measurement phase with no regions measured yet        |
    | assign_region   | Measures the shape of a region during the measurement phase       | - Determines the next region index<br>- Creates a new `RegionShape` for this region<br>- Calls the assignment closure with a mutable reference to the region shape<br>- Stores the measured shape in `regions`<br>- Returns the result of the closure | Collects information about which columns and how many rows each region uses, for planning the circuit layout |

- **AssignmentPass**: AssignmentPass handles the second phase of the V1 floor planner. Using the plan created during measurement, it assigns values to regions and tables, manages table column usage, and ensures all assignments follow the measured plan. It also handles copying values for instance constraints and filling in default values for tables.
    ```rust
    /// Assigns the circuit.
    #[derive(Debug)]
    pub struct AssignmentPass<'p, 'a, F: Field, CS: Assignment<F> + 'a> {
        plan: &'p mut V1Plan<'a, F, CS>,
        /// Counter tracking which region we need to assign next.
        region_index: usize,
    }
    ```
    - **Functions**:

    | Function Name   | Purpose/Role                                                      | Execution Logic                                                                                                  | Function in Halo2                                                      |
    |-----------------|-------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
    | new             | Creates a new `AssignmentPass`                                    | Initializes the `plan` reference and sets `region_index` to 0                                                   | Prepares the assignment phase with a fresh counter and plan reference   |
    | assign_region   | Assigns a region in the circuit                                   | - Gets the next region index<br>- Increments the region counter<br>- Enters the region in the constraint system<br>- Creates a new `V1Region`<br>- Calls the assignment closure with a mutable reference to the region<br>- Exits the region<br>- Returns the result of the closure | Performs the actual assignment of values to a region during synthesis   |
    | assign_table    | Assigns a lookup table in the circuit                             | - Enters the region in the constraint system<br>- Creates a `SimpleTableLayouter`<br>- Calls the assignment closure<br>- Exits the region<br>- Checks that all table columns have the same length<br>- Records used columns<br>- Fills in default values for unassigned cells | Ensures lookup tables are assigned correctly and consistently           |
    | constrain_instance | Constrains an advice cell to equal an instance cell            | Calls `copy` on the constraint system to link the advice cell to the instance cell, using the correct region and row offsets | Used to link advice cells to public inputs (instance columns)           |

- **V1Region**: `V1Region` is a struct that represents a single region within the V1 floor planner during circuit synthesis in Halo2. It holds a reference to the overall plan and the specific region index, and implements the RegionLayouter trait. This allows V1Region to handle all assignments, constraints, and value queries for its region, translating logical region offsets into absolute positions in the circuit according to the plan. It acts as the main interface for assigning advice, fixed, and instance values, as well as enforcing constraints within a region during the assignment phase. 
    ```rust
    struct V1Region<'r, 'a, F: Field, CS: Assignment<F> + 'a> {
        plan: &'r mut V1Plan<'a, F, CS>,
        region_index: RegionIndex,
    }
    ```
    - **Functions**:

    | Function Name                | Purpose/Role                                                      | Execution Logic                                                                                                  | Function in Halo2                                                      |
    |------------------------------|-------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
    | new                          | Creates a new V1Region instance                                   | Stores a mutable reference to the plan and the region index                                                     | Prepares a region layouter for assignments in a specific region        |
    | enable_selector              | Enables a selector at a given offset in the region                | Calls enable_selector on the constraint system with the correct absolute row offset                             | Activates a selector for constraints at a specific row in the region   |
    | assign_advice                | Assigns a value to an advice column at a given offset             | Calls assign_advice on the constraint system with the correct absolute row offset, returns a Cell               | Assigns witness values to advice columns in the region                 |
    | assign_advice_from_constant  | Assigns a constant value to an advice column                      | Calls assign_advice with a closure returning the constant, then constrains the cell to that constant            | Used for copying constants into advice columns                         |
    | assign_advice_from_instance  | Assigns a value from an instance column to an advice column       | Queries the instance value, assigns it to advice, and copies the value from instance to advice cell             | Links public inputs to advice columns in the region                    |
    | instance_value               | Queries the value of an instance column at a given row            | Calls query_instance on the constraint system                                                                   | Reads public input values for use in the region                        |
    | assign_fixed                 | Assigns a value to a fixed column at a given offset               | Calls assign_fixed on the constraint system with the correct absolute row offset, returns a Cell                | Assigns fixed values (constants) to fixed columns in the region        |
    | constrain_constant           | Constrains a cell to a constant value                            | Pushes the (constant, cell) pair to the plan's constants list                                                   | Used for enforcing constant constraints in the region                  |
    | constrain_equal              | Constrains two cells to be equal                                  | Calls copy on the constraint system to link the two cells at their absolute positions                           | Enforces equality constraints between cells in the region              |

- **synthesize of V1**: This function in Halo2 is responsible for laying out and assigning all regions, tables, and constants in a circuit using the V1 floor planning strategy. It performs a two-pass process: first, it measures the shape and requirements of each region (without assigning values), then it plans and assigns all regions and constants efficiently to avoid overlaps and ensure correct placement. This function ensures that the circuit is synthesized in a way that is both correct and optimized for the available columns, handling all region and constant assignments according to the V1 strategy. Step-by-Step Execution Flow are as follows:
    ```mermaid
    flowchart TD
        A[Create V1Plan] --> B[First Pass: Measurement]
        B --> C[Run circuit.synthesize without witnesses<br/>using MeasurementPass]
        C --> D[Planning: Region Placement]
        D --> E[Use slot_in_biggest_advice_first<br/>to sort and place regions]
        E --> F[Update plan.regions with region start positions]
        F --> G[Determine Required Rows]
        G --> H[Find highest unassigned row across all columns]
        H --> I[Plan Constant Placement]
        I --> J[For each fixed column,<br/>find available positions for constants]
        J --> K[Second Pass: Assignment]
        K --> L[Run circuit.synthesize with AssignmentPass]
        L --> M[Assign Constants]
        M --> N{Enough positions for all constants?}
        N -- No --> O[Return Error: NotEnoughColumnsForConstants]
        N -- Yes --> P[Assign each constant<br/>and copy to advice cell]
        P --> Q[Finish: Return Ok]

        style A fill:#e0eaff
        style Q fill:#d2ffd2
        style O fill:#ffd2d2
    ```
    1. **Create a V1Plan**: Initialize a new plan object that will track region starts, constants, and table columns for the circuit.
    2. **First Pass – Measurement**: Create a MeasurementPass to measure the shape (columns and rows used) of each region in the circuit. 
        * The circuit is synthesized without witnesses, using a special measurement pass to collect region information.
    3. **Planning – Region Placement**: Use the strategy module to determine how to position regions in the circuit.
        * The `slot_in_biggest_advice_first` function sorts and places regions to minimize column contention and overlap.
        * The plan’s region start positions are updated.
    4. **Determine Required Rows**: Calculate the total number of rows needed for the planned circuit by finding the highest unassigned row across all columns.
    5. **Plan Constant Placement**: For each fixed column, determine where constants can be placed within the assigned rows, using the column allocations from the planning step.
    6. **Second Pass – Assignment**: Create an AssignmentPass to actually assign values to regions and tables according to the plan.
        * The circuit is synthesized again, this time with real assignments.
    7. **Assign Constants**: Check if there are enough positions for all constants.
        * If not, return an error.
        * Otherwise, assign each constant to its planned position and copy its value to the appropriate advice cell.
    8. **Finish**: Return Ok(()) if everything succeeded.

#### 7. Expression
```mermaid
classDiagram
    class Expression~F~ {
        <<enum>>
        +Constant（F）
        +Selector（Selector）
        +Fixed（FixedQuery）
        +Advice（AdviceQuery）
        +Instance（InstanceQuery）
        +Negated（Box~Expression< F>~）
        +Sum（Box~Expression< F>~, Box~Expression< F>~）
        +Product（Box~Expression< F>~, Box~Expression< F>~）
        +Scaled（Box~Expression< F>~, F）

        +evaluate(...) T
        +degree() usize
        +square() Self 
        +contains_simple_selector() bool 
        +extract_simple_selector() Option~Selector~
    }

    class Selector {
        index: usize
        is_simple: bool
    }

    class FixedQuery {
        index: usize
        column_index: usize
        rotation: Rotation
    }

    class AdviceQuery {
        index: usize
        column_index: usize
        rotation: Rotation
    }

    class InstanceQuery {
        index: usize
        column_index: usize
        rotation: Rotation
    }

    class Rotation {
        offset: i32
    }

    Expression~F~ *-- Selector : contains
    Expression~F~ *-- FixedQuery : contains
    Expression~F~ *-- AdviceQuery : contains
    Expression~F~ *-- InstanceQuery : contains
    FixedQuery *-- Rotation : contains
    AdviceQuery *-- Rotation : contains
    InstanceQuery *-- Rotation : contains

    Expression~F~ ..|> Debug : implements
    Expression~F~ ..|> Neg : implements
    Expression~F~ ..|> Add : implements
    Expression~F~ ..|> Sub : implements
    Expression~F~ ..|> Mul : implements
```
- **Expression** : a fundamental component in Halo2's constraint system, representing low-degree polynomial expressions that define the relationships between circuit elements. `Expression` plays a central role in translating high-level circuit specifications into the low-level polynomial constraints required for efficient and secure zero-knowledge proofs.
    * This enum defines all the possible forms a low-degree polynomial expression can take in the circuit system, including constants, column queries, and arithmetic operations.
    ```rust
    /// Low-degree expression representing an identity that must hold over the committed columns.
    #[derive(Clone)]
    pub enum Expression<F> {
        /// This is a constant polynomial
        Constant(F),
        /// This is a virtual selector
        Selector(Selector),
        /// This is a fixed column queried at a certain relative location
        Fixed(FixedQuery),
        /// This is an advice (witness) column queried at a certain relative location
        Advice(AdviceQuery),
        /// This is an instance (external) column queried at a certain relative location
        Instance(InstanceQuery),
        /// This is a negated polynomial
        /// The use of Box allows for recursive expressions (expressions containing other expressions). 
        Negated(Box<Expression<F>>),
        /// This is the sum of two polynomials
        Sum(Box<Expression<F>>, Box<Expression<F>>),
        /// This is the product of two polynomials
        Product(Box<Expression<F>>, Box<Expression<F>>),
        /// This is a scaled polynomial
        /// Represents an expression scaled (multiplied) by a constant value of type F.
        Scaled(Box<Expression<F>>, F),
    }
    ```
- **Function**:
    | Function | Description | Principle in Halo2 |
    | --- | --- | --- |
    | `evaluate(...)` | This method recursively evaluates an `Expression<F>` by applying the provided closures (function arguments) to each variant of the enum. It is generic over the return type T, so you can use it to evaluate to any type you want (for example, a field element, a symbolic expression, or a string).<br><br>  For composite expressions (like sum, product, negated, scaled), the function recursively evaluates sub-expressions and then applies the corresponding operation closure. | This function is a visitor for the Expression<F> enum. It walks the expression tree and applies the provided closures to each node, allowing you to define custom logic for each kind of expression and operation. It is a powerful pattern for symbolic computation, interpretation, or code generation. |
    | `degree()` | Computes the degree of the polynomial expression.<br><br> Constant:0<br> Selector:1<br> Fixed:1<br> Advice:1<br> Instance:1<br> Negated(poly):poly.degree()<br> Sum(a, b):max(a.degree(), b.degree())<br> Product(a, b):a.degree() + b.degree()<br> Scaled(poly, _):poly.degree() | The degree is crucial for determining the size of the polynomial domain and ensuring the circuit's constraints fit within the allowed computational limits. |
    | `square()` | Returns the square of the expression. | Convenience method for common polynomial operations, simplifying circuit design. |
    | `contains_simple_selector()` | Checks whether or not this expression contains a simple `Selector`.<br><br> This function uses `evaluate` to recursively walk the expression tree and returns true if any part of the expression contains a simple selector, by checking each selector node with selector.is_simple(). | Simple selectors have restrictions on their use in expressions to maintain circuit efficiency and correctness. |
    | `extract_simple_selector()` | This function recursively traverses the expression tree using the `evaluate` method to find and extract a simple selector, if present.<br><br> For each selector node, it checks if the selector is simple—if so, it returns `Some(selector)`, otherwise `None`. For `sum` and `product` nodes, it uses a custom operator (op) that returns the found `selector` if only one is present, but panics if two simple selectors are found in the same expression (which is not allowed). If no simple selector is found, it returns `None`. | Used during circuit optimization to handle simple selectors separately from other expression components. |
    | `fmt(...)` | Formats the expression for debugging purposes. | Helps developers inspect and debug expressions during circuit development. |
    | `neg()` | Implements the negation operator for expressions.<br><br> -expr becomes Expression::Negated(Box::new(expr)). | Allows the creation of negative polynomial terms. |
    | `add(rhs)` | Implements the addition operator for expressions.<br><br> If either side contains a simple selector, it panics. Otherwise, returns Expression::Sum(Box::new(self), Box::new(rhs)). | Creates a new expression representing the sum of two polynomials, ensuring no simple selectors are used in the operation. |
    | `sub(rhs)` | Implements the subtraction operator for expressions.<br><br> Panics if either side contains a simple selector. Otherwise, returns the sum of self and the negation of rhs (Expression::Sum(Box::new(self), Box::new(-rhs))). | Creates a new expression representing the difference of two polynomials, ensuring no simple selectors are used in the operation. |
    | `mul(rhs)` | Implements the multiplication operator for expressions.<br><br> Panics if both sides contain a simple selector. Otherwise, returns Expression::Product(Box::new(self), Box::new(rhs)). | Creates a new expression representing the product of two polynomials, preventing the multiplication of two expressions containing simple selectors. |
    | `mul(rhs: F)` | Implements scalar multiplication for expressions.<br><br> Implements the Mul trait for multiplying an expression by a constant of type F. Returns Expression::Scaled(Box::new(self), rhs). | Scales the polynomial by a constant field element, simplifying coefficient adjustments in constraints. |    

#### 8. VirtualCell and VirtualCells
```mermaid
classDiagram
    class VirtualCell {
        column: Column~Any~
        rotation: Rotation
        +from((Col, Rotation)) Self
    }

    class VirtualCells~'a, F~ {
        meta &'a mut ConstraintSystem~F~
        queried_selectors Vec~Selector~
        queried_cells Vec~VirtualCell~
        +new(meta) Self
        +query_selector(selector) Expression~F~
        +query_fixed(column) Expression~F~
        +query_advice(column, at) Expression~F~
        +query_instance(column, at) Expression~F~
        +query_any(column, at) Expression~F~
    }

    class ConstraintSystem~F~ {
        <<Incomplete description>>
        +query_fixed_index(column)
        +query_advice_index(column, at)
        +query_instance_index(column, at)
    }

    class Expression~F~ {
        <<enum: Incomplete description>>
    }

    class Column~C~ {
        index: usize
        column_type: C
    }

    class Rotation {
        offset: i32
    }

    class Selector {
        index: usize
        is_simple: bool
    }

    VirtualCell "1" *-- "1" Column~Any~ : contains
    VirtualCell "1" *-- "1" Rotation : contains
    VirtualCells "1" *-- "*" VirtualCell : contains
    VirtualCells "1" *-- "*" Selector : contains
    VirtualCells "1" o-- "1" ConstraintSystem~F~ : uses
    VirtualCells "1" -- "1" Expression~F~ : returns
```
- **VirtualCell**: a PLONK cell that has been queried at a particular relative offset (column and rotation) within a custom gate. It also provides a convenient way to create a `VirtualCell` from any column type and a rotation using the `From` trait.
- **VirtualCells**: The `VirtualCells` struct is a utility for querying "virtual cells" when defining custom gates or lookup tables in a Halo2 circuit. 
    * a collection of `VirtualCell` instances that can be queried within a gate. It provides methods to query selectors, fixed columns, advice columns, and instance columns at specific rotations.
    ```rust
    /// Exposes the "virtual cells" that can be queried while creating a custom gate or lookup table.
    #[derive(Debug)]
    pub struct VirtualCells<'a, F: Field> {
        meta: &'a mut ConstraintSystem<F>,
        /// Keeps track of all selectors queried during gate construction.
        queried_selectors: Vec<Selector>,
        /// Keeps track of all cells (columns and rotations) queried.
        queried_cells: Vec<VirtualCell>,
    }
    ``` 

* **Functions**:
    | Function | Description | Principle in Halo2 |
    | --- | --- | --- |
    | `VirtualCell::from((Col, Rotation))` | Creates a `VirtualCell` from a column and rotation. | Encapsulates a column and rotation into a single queryable cell for use in gates. |
    | `VirtualCells::new(meta)` | Initializes a new `VirtualCells` instance with a reference to `ConstraintSystem`. | Sets up the context for querying cells and selectors within a gate. |
    | `query_selector(selector)` | Queries a selector at the current position and records it. | Selectors enable conditional enforcement of gates, critical for circuit flexibility. |
    | `query_fixed(column)` | Queries a fixed column at the current rotation and records the cell. | Fixed columns store constants or precomputed values, queried for constraint evaluation. |
    | `query_advice(column, at)` | Queries an advice column at a specified rotation and records the cell. | Advice columns hold private witness data, queried to enforce constraints on prover-provided values. |
    | `query_instance(column, at)` | Queries an instance column at a specified rotation and records the cell. | Instance columns store public inputs, queried to ensure consistency with verifier's expectations. |
    | `query_any(column, at)` | Queries a column of any type (advice, fixed, instance) with rotation checks. | Provides a generic interface for querying columns, ensuring fixed columns are only queried at the current rotation.<br><br> **Reason**<br><br> **Fixed columns are not witness-dependent**: Their values are known and fixed for all rows, so there is no meaningful concept of "previous" or "next" for a fixed column.<br> **Implementation constraint**: Allowing rotations on fixed columns would complicate the constraint system and could lead to ambiguity or errors, since the value at any rotation is always the same as the value at the current row.<br> **Safety**: By restricting queries to only the current row (Rotation::cur()), the API prevents accidental misuse or logical errors in circuit design.<br><br> You can only query a fixed column at the current row because its value does not change across rows, and allowing other rotations would not make sense and could introduce bugs or confusion in the circuit logic. That’s why the code panics if you try to query a fixed column at any rotation other than the current one. |

* **Role in Halo2**:
    | Role | Description |
    | --- | --- |
    | Abstraction for Querying | `VirtualCell` represents a cell (column + rotation) queried within a gate.<br><br> `VirtualCells` provides methods to query columns and selectors, building up the list of dependencies for a gate. |
    | Constraint Definition | By querying columns and selectors, circuits define polynomial constraints that must hold true.<br><br> These constraints are critical for proving that the circuit's computations are correct without revealing private inputs. |
    | Circuit Flexibility | Rotations allow querying columns in different rows, enabling temporal relationships (e.g., state transitions).<br><br> Selectors enable conditional constraints, allowing parts of the circuit to be activated/deactivated based on computation needs. |
    | Integration with `ConstraintSystem` | `VirtualCells` interacts with `ConstraintSystem` to manage column indices and track dependencies, ensuring efficient constraint enforcement during proof generation. |


#### 9. Gate and PinnedGates
```mermaid
classDiagram
    class Gate~F: Field~ {
        name: &'static str
        constraint_names: Vec<&'static str>
        polys: Vec~Expression< F>~
        queried_selectors: Vec~Selector~
        queried_cells: Vec~VirtualCell~
        +name() &'static str
        +constraint_name(constraint_index: usize) &'static str
        +polynomials() &[Expression~F~]
        +queried_selectors() &[Selector]
        +queried_cells() &[VirtualCell]
    }

    class PinnedGates~'a, F~ {
        gates: &'a Vec< Gate~F~>
        +fmt(f: &mut Formatter): Result<（）, Error>
    }

    class Expression~F~ {
        <<enum>>
    }

    class Selector {
        index: usize
        is_simple: bool
    }

    class VirtualCell {
        column: Column~Any~
        rotation: Rotation
    }

    class Column~C~ {
        index: usize
        column_type: C
    }

    class Rotation {
        offset: i32
    }

    Gate~F~ "1" -- "*" Expression~F~ : contains
    Gate~F~ "1" -- "*" Selector : contains
    Gate~F~ "1" -- "*" VirtualCell : contains
    PinnedGates~'a, F~ "1" -- "*" Gate~F~ : references
    VirtualCell "1" -- "1" Column~Any~ : contains
    VirtualCell "1" -- "1" Rotation : contains
```
- **Gate**: Represents a logical gate in a circuit, which enforces constraints between different cells (columns and rows). Gates are fundamental building blocks in zero-knowledge proofs, ensuring that computed values adhere to specific relationships.
    ```rust
    #[derive(Clone, Debug)]
    pub(crate) struct Gate<F: Field> {
        /// The name of the gate
        name: &'static str,
        /// A vector of names for each constraint in the gate.
        constraint_names: Vec<&'static str>,
        /// A vector of polynomial expressions representing the constraints.
        polys: Vec<Expression<F>>,
        /// We track queried selectors separately from other cells, so that we can use them to
        /// trigger debug checks on gates.
        /// Selectors used in the gate
        queried_selectors: Vec<Selector>,
        /// Cells (columns and rotations) that are queried in this gate.
        queried_cells: Vec<VirtualCell>,
    }
    ```
- **PinnedGates**: PinnedGates serves as a lightweight, immutable wrapper around a collection of Gate instances within a `ConstraintSystem`. Its primary role is to provide a debug-friendly view of the gates' polynomial constraints while ensuring that the underlying data remains unchanged. By encapsulating references to the original gates (&'a Vec<Gate<F>>), PinnedGates avoids data duplication and maintains consistency with the constraint system's state.
    * The Debug implementation of `PinnedGates` formats the gate polynomials for logging and inspection, which is crucial for developers during circuit development and testing. This allows them to verify that the constraints defined in the gates (e.g., arithmetic relationships between columns) are correctly formulated. 
    * Additionally, `PinnedGates` is integrated into the `PinnedConstraintSystem`, which represents the minimal set of parameters needed to determine the constraint system's behavior. This integration ensures that developers can access and analyze the gate constraints without modifying the original constraint system, promoting safety and efficiency in the circuit design process.

    | Function | Description | Role in Halo2 |
    | --- | --- | --- |
    | `Gate::name()` | Returns the static string identifier for the gate. | Used for debugging and logging to identify gates in the circuit. |
    | `Gate::constraint_name(constraint_index: usize)` | Returns the name of a specific constraint within the gate. | Helps in debugging by providing human-readable names for individual constraints. |
    | `Gate::polynomials()` | Returns the polynomial expressions that define the gate's constraints. | Essential for verifying that the circuit adheres to the specified mathematical relationships during proof generation and verification. |
    | `Gate::queried_selectors()` | Returns the selectors used to enable or disable the gate in specific rows. | Selectors control when a gate's constraints are active, allowing conditional logic in the circuit. |
    | `Gate::queried_cells()` | Returns the cells (columns and rotations) involved in the gate's constraints. | Helps in understanding which parts of the circuit are affected by the gate's constraints. |
    | `PinnedGates::fmt()` | Formats the gate polynomials for debugging purposes. | Used by developers to inspect the polynomial expressions during circuit development and testing. |

#### 10. Constraint and Constraints
```mermaid
classDiagram
    class Constraint~F~ {
        name: &'static str
        poly: Expression~F~
        +from(Expression~F~) Self
        +from((&'static str, Expression~F~)) Self
    }

    class Constraints~F, C, Iter~ {
        selector: Expression~F~
        constraints: Iter
        +with_selector(selector, constraints) Self
    }

    class Expression~F~ {
        <<enum: Incomplete definition>>
    }

    Constraint~F~ "1" *-- "1" Expression~F~ : contains
    Constraints~F, C, Iter~ "1" *-- "1" Expression~F~ : contains
    Constraints~F, C, Iter~ "1" *-- "1" Iter : contains
    Constraints~F, C, Iter~ ..|> IntoIterator : implements

    Constraint~F~ ..|> From~Expression< F>~ : implements
    Vec~Constraint< F>~ ..|> From~Expression<F>~ : implements

    ApplySelectorToConstraint~F, C~ ..> Constraint~F~ : returns
    ConstraintsIterator~F, C, I~ ..> ApplySelectorToConstraint~F, C~ : uses
```
- **Constraint**: Represents a single constraint in a circuit, which is defined by a polynomial expression. `Constraints` are used to enforce relationships between different cells (columns and rows) in the circuit, ensuring that the computed values adhere to specific mathematical rules.
    ```rust
    /// An individual polynomial constraint.
    ///
    /// These are returned by the closures passed to `ConstraintSystem::create_gate`.
    #[derive(Debug)]
    pub struct Constraint<F: Field> {
        name: &'static str,
        poly: Expression<F>,
    }
    ```
- **Constraints**: A collection of constraints that can be applied to a circuit, typically used to group multiple constraints under a common selector expression. This allows for conditional enforcement of constraints based on the selector's activation.
    ```rust
    #[derive(Debug)]
    pub struct Constraints<F: Field, C: Into<Constraint<F>>, Iter: IntoIterator<Item = C>> {
        selector: Expression<F>,
        constraints: Iter,
    }
    ```

- **ConstraintsIterator**: An iterator type for `Constraints`, which applies the selector to each constraint during iteration. This allows for efficient processing of constraints in the circuit, ensuring that only active constraints are evaluated based on the selector(lazily applying the selector during iteration). 
    * It zips (pairs together) an infinite iterator that repeats the selector expression with another iterator I (which yields constraints).
    * Then it maps each pair (selector, constraint) using the apply_selector_to_constraint function, producing a Constraint<F> for each pair.
    ```rust
    type ApplySelectorToConstraint<F, C> = fn((Expression<F>, C)) -> Constraint<F>;
    type ConstraintsIterator<F, C, I> = std::iter::Map<
        std::iter::Zip<std::iter::Repeat<Expression<F>>, I>,
        ApplySelectorToConstraint<F, C>,
    >;
    ```

- **Functions**:

    | Function | Description | Principle in Halo2 |
    | --- | --- | --- |
    | `Constraint::from(Expression<F>)` | Converts an `Expression` into a `Constraint` with an empty name. | Simplifies constraint creation for unnamed expressions. |
    | `Constraint::from((&'static str, Expression<F>))` | Creates a named `Constraint` from an `Expression`. | Enhances debuggability by associating names with constraints. |
    | `Constraints::with_selector(selector, constraints)` | Constructs a `Constraints` instance with a selector and iterator of constraints.<br><br> Each constraint `c` in `constraints` (iterator) will be converted into the constraint `selector * c`, so the selector controls when the constraints are active. | Binds constraints to a selector, forming `selector * constraint` relationships. |
    | **`apply_selector_to_constraint`** | Applies a selector to a constraint, producing `selector * constraint.poly`. | Implements the core logic for combining selectors with constraints. |
    | **`IntoIterator for Constraints`** | Converts `Constraints` into an iterator of `Constraint` instances. | This implementation allows a Constraints object to be turned into an iterator that yields each constraint multiplied by the common selector.<br><br> This is useful for applying a selector to a whole set of constraints in a circuit, enabling or disabling them together. |

- **Example**:
    ```rust
    /// A set of polynomial constraints with a common selector.

    use group::ff::Field;
    use halo2_proofs::{pasta::Fp, plonk::{Constraints, Expression}, poly::Rotation};
    use halo2_proofs::plonk::ConstraintSystem;

    let mut meta = ConstraintSystem::<Fp>::default();
    /// Three advice columns (a, b, c) and one selector (s) are allocated.
    let a = meta.advice_column();
    let b = meta.advice_column();
    let c = meta.advice_column();
    let s = meta.selector();
    /// defines a new gate named "foo".
    meta.create_gate("foo", |meta| {
        /// Inside the closure, the circuit designer can query columns and selectors at specific rotations (row offsets).
        /// Queries the value of column a at the next row.
        let next = meta.query_advice(a, Rotation::next());
        let a = meta.query_advice(a, Rotation::cur());
        let b = meta.query_advice(b, Rotation::cur());
        let c = meta.query_advice(c, Rotation::cur());
        /// Queries the selector at the current row.
        let s_ternary = meta.query_selector(s);
        /// Defines an expression representing 1 - a
        let one_minus_a = Expression::Constant(Fp::ONE) - a.clone();
        /// creates a set of constraints that are only active when the selector is enabled.
        Constraints::with_selector(
            s_ternary,
            std::array::IntoIter::new([
                /// Ensures that a is either 0 or 1 by enforcing a * (1 - a) == 0.
                ("a is boolean", a.clone() * one_minus_a.clone()),
                /// Enforces that the value of a in the next row equals b if a is 1, or c if a is 0. This is written as next - (a * b + (1 - a) * c) == 0.
                ("next == a ? b : c", next - (a * b + one_minus_a * c)),
            ]),
        )
    });
    ```

#### 11. ConstraintSystem and PinnedConstraintSystem
```mermaid
classDiagram
    class ConstraintSystem {
        +num_fixed_columns: usize
        +num_advice_columns: usize
        +num_instance_columns: usize
        +num_selectors: usize
        +selector_map: Vec< Column< Fixed>>
        +gates: Vec< Gate< F>>
        +advice_queries: Vec<（Column< Advice>, Rotation）>
        +num_advice_queries: Vec< usize>
        +instance_queries: Vec<（Column< Instance>, Rotation）>
        +fixed_queries: Vec<（Column< Fixed>, Rotation）>
        +permutation: permutation::Argument
        +lookups: Vec< lookup::Argument< F>>
        +constants: Vec< Column< Fixed>>
        +minimum_degree: Option< usize>
        +default() ConstraintSystem< F>
        +pinned() pinnedConstraintSystem<'_, F>
        +enable_constant(column: Column< Fixed>)
        +enable_equality(column: C)
        +lookup(table_map) usize
        +query_fixed_index(column: Column< Fixed>) usize
        +query_advice_index(column: Column< Advice>, at: Rotation) usize
        +query_instance_index(column: Column< Instance>, at: Rotation) usize
        +query_any_index(column: Column< Any>, at: Rotation) usize
        +get_advice_query_index(column: Column< Advice>, at: Rotation) usize
        +get_fixed_query_index(column: Column< Fixed>, at: Rotation) usize
        +get_instance_query_index(column: Column< Instance>, at: Rotation) usize
        +get_any_query_index(column: Column< Any>) usize
        +set_minimum_degree(degree: usize)
        +create_gate(name: &'static str, constraints)
        +compress_selectors(selectors: Vec< Vec< bool>>) （Self, Vec< Vec< F>>）
        +selector() Selector
        +complex_selector() Selector
        +lookup_table_column() TableColumn
        +fixed_column() Column< Fixed>
        +advice_column() Column< Advice>
        +instance_column() Column< Instance>
        +degree() usize
        +blinding_factors() usize
        +minimum_rows() usize
    }

    class PinnedConstraintSystem {
        +num_fixed_columns: &'a usize
        +num_advice_columns: &'a usize
        +num_instance_columns: &'a usize
        +num_selectors: &'a usize
        +gates: PinnedGates<'a, F>
        +advice_queries: &'a Vec<（Column< Advice>, Rotation）>
        +instance_queries: &'a Vec<（Column< Instance>, Rotation）>
        +fixed_queries: &'a Vec<（Column< Fixed>, Rotation）>
        +permutation: &'a permutation::Argument
        +lookups: &'a Vec< lookup::Argument< F>>
        +constants: &'a Vec< Column< Fixed>>
        +minimum_degree: &'a Option< usize>
    }

    ConstraintSystem --> PinnedConstraintSystem : creates

    class Gate {
        +name: str
        +constraint_names: str_list
        +polys: Expression_list
        +queried_selectors: Selector_list
        +queried_cells: VirtualCell_list
    }

    class Column_Fixed {
        +index: usize
        +column_type: Fixed
    }

    class Column_Advice {
        +index: usize
        +column_type: Advice
    }

    class Column_Instance {
        +index: usize
        +column_type: Instance
    }

    class Selector {
        +index: usize
        +is_simple: bool
    }

    class TableColumn {
        +inner: Column< Fixed>
    }

    class Expression {
        +degree()
        +extract_simple_selector()
        +contains_simple_selector()
        +evaluate()
    }

    class Rotation {
        +0: i32
    }

    class VirtualCell {
        +column: Column_Any
        +rotation: Rotation
    }

    class VirtualCells {
        +queried_selectors: Selector_list
        +queried_cells: VirtualCell_list
    }

    %% Associations
    ConstraintSystem --> Gate : has
    ConstraintSystem --> Column_Fixed : has
    ConstraintSystem --> Column_Advice : has
    ConstraintSystem --> Column_Instance : has
    ConstraintSystem --> Selector : has
    ConstraintSystem --> TableColumn : has
    ConstraintSystem --> Expression : uses
    ConstraintSystem --> Rotation : uses
    ConstraintSystem --> VirtualCells : uses
```
- **ConstraintSystem struct**: The `ConstraintSystem` trait plays a central role in Halo2 by providing a framework for defining, managing, and optimizing the constraints of a circuit, ensuring the correct assignment of values and the enforcement of logical relationships within the circuit.
    ```rust
    /// This is a description of the circuit environment, such as the gate, column and
    /// permutation arrangements.
    #[derive(Debug, Clone)]
    pub struct ConstraintSystem<F: Field> {
        pub(crate) num_fixed_columns: usize,
        pub(crate) num_advice_columns: usize,
        pub(crate) num_instance_columns: usize,
        pub(crate) num_selectors: usize,

        /// This is a cached vector that maps virtual selectors to the concrete
        /// fixed column that they were compressed into. This is just used by dev
        /// tooling right now.
        pub(crate) selector_map: Vec<Column<Fixed>>,

        pub(crate) gates: Vec<Gate<F>>,
        pub(crate) advice_queries: Vec<(Column<Advice>, Rotation)>,
        // Contains an integer for each advice column
        // identifying how many distinct queries it has
        // so far; should be same length as num_advice_columns.
        num_advice_queries: Vec<usize>,
        pub(crate) instance_queries: Vec<(Column<Instance>, Rotation)>,
        pub(crate) fixed_queries: Vec<(Column<Fixed>, Rotation)>,

        // Permutation argument for performing equality constraints
        pub(crate) permutation: permutation::Argument,

        // Vector of lookup arguments, where each corresponds to a sequence of
        // input expressions and a sequence of table expressions involved in the lookup.
        pub(crate) lookups: Vec<lookup::Argument<F>>,

        // Vector of fixed columns, which can be used to store constant values
        // that are copied into advice columns.
        pub(crate) constants: Vec<Column<Fixed>>,

        pub(crate) minimum_degree: Option<usize>,
    }
    ```
- **PinnedConstraintSystem struct**: Represents the minimal parameters that determine a `ConstraintSystem`.
    ```rust
    #[allow(dead_code)]
    #[derive(Debug)]
    pub struct PinnedConstraintSystem<'a, F: Field> {
        num_fixed_columns: &'a usize,
        num_advice_columns: &'a usize,
        num_instance_columns: &'a usize,
        num_selectors: &'a usize,
        gates: PinnedGates<'a, F>,
        advice_queries: &'a Vec<(Column<Advice>, Rotation)>,
        instance_queries: &'a Vec<(Column<Instance>, Rotation)>,
        fixed_queries: &'a Vec<(Column<Fixed>, Rotation)>,
        permutation: &'a permutation::Argument,
        lookups: &'a Vec<lookup::Argument<F>>,
        constants: &'a Vec<Column<Fixed>>,
        minimum_degree: &'a Option<usize>,
    }
    ```    
- **Difference**:   

    | Struct | Description |
    | --- | --- |
    | `ConstraintSystem` | **Mutable Configuration**:<br> `ConstraintSystem` is designed to be a mutable and dynamic structure. It allows circuit designers to configure various aspects of the circuit during the circuit definition phase. For example, designers can add gates, columns, selectors, permutation arguments, and lookup arguments. The fields in `ConstraintSystem` like `num_fixed_columns`, `num_advice_columns`, etc., can be modified as the circuit is being defined. This mutability is crucial for building complex circuits where different components need to be added and adjusted according to the specific requirements of the problem.<br><br> **Caching and Intermediate State**:<br> It also contains some cached information such as `selector_map`, which is used by development tooling. This caching can improve the efficiency of certain operations during the circuit development and debugging process. |
    | `PinnedConstraintSystem` | **Immutability and Reference**: `PinnedConstraintSystem` is an immutable view of the `ConstraintSystem`. It holds references to the data in the `ConstraintSystem` rather than owning the data itself. This immutability is useful when you want to pass around a snapshot of the circuit's constraint configuration without allowing any modifications. For example, in the verification process of a zero - knowledge proof, you don't want the constraint system to be changed accidentally. By using `PinnedConstraintSystem`, you can ensure that the constraints remain consistent throughout the verification.<br><br> **Lifetime Management**: The use of references in `PinnedConstraintSystem` also helps with lifetime management. It allows the `PinnedConstraintSystem` to be used within a specific scope without the need to clone the entire `ConstraintSystem`, which can be expensive in terms of memory and performance. |

- **Functions**: 

    | Function Name | Functionality | Principle in Halo2 |
    | --- | --- | --- |
    | `default()` | Returns a default instance of `ConstraintSystem<F>`. | Initializes all fields of the constraint system to their default values, providing a starting point for circuit configuration. |
    | `pinned()` | Obtains a pinned version of the constraint system with minimal parameters (`PinnedConstraintSystem`).<br><br> Returns a "pinned" (immutable, reference-based) version of the constraint system, which contains only references to the essential configuration data. This is useful for passing around a lightweight, read-only view of the constraint system. | The pinned version is used to represent the essential information of the constraint system, which can be useful for serialization or passing a lightweight version of the system. |
    | `get_advice_query_index(column: Column<Advice>, at: Rotation)` | Returns the index of an existing advice query. Panics if the query doesn't exist. | Used to retrieve the index of an advice query when it is known to exist. |
    | `get_fixed_query_index(column: Column<Fixed>, at: Rotation)` | Returns the index of an existing fixed query. Panics if the query doesn't exist. | Similar to `get_advice_query_index`, but for fixed queries. |
    | `get_instance_query_index(column: Column<Instance>, at: Rotation)` | Returns the index of an existing instance query. Panics if the query doesn't exist. | Similar to `get_advice_query_index`, but for instance queries. |
    | `get_any_query_index(column: Column<Any>)` | Determines the column type and calls the appropriate get query index function. | Another helper function for retrieving query indices. |
    | `query_fixed_index(column: Column<Fixed>)` | Returns the index of a fixed column query at the current rotation. If the query doesn't exist, it creates a new one. | This function helps in managing and indexing fixed queries, which are used to access values in fixed columns during circuit evaluation. |
    | `query_advice_index(column: Column<Advice>, at: Rotation)` | Returns the index of an advice query for a given column and rotation. If the query doesn't exist, it creates a new one. | Similar to `query_fixed_index`, but for advice columns. Advice columns are used to store private values in the circuit. |
    | `query_instance_index(column: Column<Instance>, at: Rotation)` | Returns the index of an instance query for a given column and rotation. If the query doesn't exist, it creates a new one. | Instance columns are used to store public values in the circuit, and this function helps in managing queries to these columns. |
    | `query_any_index(column: Column<Any>, at: Rotation)` | Determines the column type and calls the appropriate query index function. | This is a helper function that simplifies the process of querying different types of columns. |
    | `enable_equality<C: Into<Column<Any>>>(column: C)` | Enables the ability to enforce equality over cells in a given column. | This function adds the column to the permutation argument, which is used to enforce equality constraints between cells in different columns or rows. |
    | `enable_constant(column: Column<Fixed>)` | Enables a fixed column for global constant assignments and enables equality on it. | By marking a column as a fixed column and enabling equality, the system can ensure that the values in this column are consistent across different parts of the circuit. |
    | `selector()` | Allocate a new (simple) selector. Simple selectors cannot be added to expressions nor multiplied by other expressions containing simple selectors. Also, simple selectors may not appear in lookup argument inputs.<br><br> Reason: TODO(sure2web3) | Simple selectors are used to enable or disable certain parts of the circuit, but they have restrictions on how they can be used in expressions. |
    | `complex_selector()` | Allocates a new complex selector that can appear anywhere in expressions. | Complex selectors provide more flexibility in circuit design compared to simple selectors. |
    | `fixed_column()` | Allocates a new fixed column. | Fixed columns are used to store constant values or values that are known in advance. |
    | `lookup_table_column()` | Allocates a new fixed column for a lookup table. | Lookup tables are used to store pre-defined values that can be accessed during circuit evaluation. |
    | `advice_column()` | Allocates a new advice column. | Advice columns are used to store private values that are provided by the prover. |
    | `instance_column()` | Allocates a new instance column. | Instance columns are used to store public values that are known to both the prover and the verifier. |
    | `set_minimum_degree(degree: usize)` | Sets the minimum degree required by the circuit, which can be set to a larger amount than actually needed. | This can be used to force the permutation argument to involve more columns in the same set, which can affect the performance and complexity of the circuit. |
    | `degree()` | Computes the degree of the constraint system, which is the maximum degree of all constraints.<br><br> This function computes the maximum degree required by the constraint system by taking the highest degree among the permutation argument, all lookup arguments, all gate polynomials, and the user-specified minimum degree. This ensures the system uses a large enough domain for all constraints. | The degree of the constraint system is important for determining the size of the polynomial domain and the complexity of the circuit. |
    | `lookup(table_map: impl FnOnce(&mut VirtualCells<'_, F>) -> Vec<(Expression<F>, TableColumn)>) -> usize` | Adds a lookup argument for input expressions and table columns.<br><br> `table_map` returns a map between input expressions and the table columns they need to match.<br><br> This function adds a new lookup argument to the constraint system by mapping input expressions to table columns, ensuring no simple selectors are used in the inputs, and registering the lookup for later use in the circuit. It returns the index of the newly added lookup argument. | Lookup arguments are used to check if certain values in the input expressions match values in the table columns, which is useful for implementing complex operations like range checks. |
    | `create_gate<C: Into<Constraint<F>>, Iter: IntoIterator<Item = C>>(name: &'static str, constraints: impl FnOnce(&mut VirtualCells<'_, F>) -> Iter)` | Creates a new gate with the given name and constraints. This method will panic if `constraints` returns an empty iterator, because Gates must contain at least one constraint.<br><br> This function creates a new gate in the constraint system. It takes a name and a closure that generates constraints using a `VirtualCells` context. The closure is called to produce the constraints, and the function collects the names and polynomial expressions from them. It also records which selectors and cells were queried. The function asserts that at least one constraint is provided, then adds a new Gate with all this information to the system. This ensures each gate is well-defined and tracks all relevant queries for circuit synthesis. | Gates are used to define the logical rules and constraints of the circuit. Each gate contains one or more polynomial constraints. |
    | `compress_selectors(selectors: Vec<Vec<bool>>)` | Compresses selectors together based on their assignments, adds new fixed columns, and returns the polynomials for those columns.<br><br> TODO(sure2web3): This function depends on `compress_selectors` | Selector compression helps in optimizing the use of fixed columns by combining multiple selectors into a single column, reducing the overall circuit size.<br><br> The `compress_selectors` function optimizes the use of selectors in a Halo2 circuit. Selectors are boolean flags (per row) that enable or disable constraints (gates). This function: Compresses multiple selectors into a minimal set of fixed columns. Updates the constraint system to use these new columns. Returns the updated constraint system and the polynomials for the new selector columns. This is important for efficiency and for ensuring the circuit’s structure matches the actual selector usage.<br><br> This will compress selectors together depending on their provided assignments. This `ConstraintSystem` will then be modified to add new fixed columns (representing the actual selectors) and will return the polynomials for those columns. Finally, an internal map is updated to find which fixed column corresponds with a given `Selector`. |
    | `blinding_factors()` | Computes the number of blinding factors necessary to perfectly blind the prover's witness polynomials. | Blinding factors are random values added to the witness polynomials to ensure zero-knowledge.<br><br>Blinding factors are used to protect the privacy of the prover's witness values during the proof generation process. |
    | `minimum_rows()` | Returns the minimum number of rows needed to account for blinding factors and other requirements. | This value is used to ensure that the circuit has enough rows to accommodate all the necessary computations and constraints.<br><br> The minimum number of rows ensures there is enough space in the circuit for all blinding factors, special polynomial roles, and at least one usable row for computation. This is critical for the soundness and security of the proof system. |

#### 12. Circuit
- **Circuit Trait**: This is a trait that circuits provide implementations for so that the backend prover can ask the circuit to synthesize using some given [`ConstraintSystem`] implementation.
    ```rust
    pub trait Circuit<F: Field> {
        /// This is a configuration object that stores things like columns.
        type Config: Clone;
        /// The floor planner used for this circuit. This is an associated type of the
        /// `Circuit` trait because its behaviour is circuit-critical.
        type FloorPlanner: FloorPlanner;

        /// Returns a copy of this circuit with no witness values (i.e. all witnesses set to
        /// `None`). For most circuits, this will be equal to `Self::default()`.
        fn without_witnesses(&self) -> Self;

        /// The circuit is given an opportunity to describe the exact gate
        /// arrangement, column arrangement, etc.
        fn configure(meta: &mut ConstraintSystem<F>) -> Self::Config;

        /// Given the provided `cs`, synthesize the circuit. The concrete type of
        /// the caller will be different depending on the context, and they may or
        /// may not expect to have a witness present.
        fn synthesize(&self, config: Self::Config, layouter: impl Layouter<F>) -> Result<(), Error>;
    }
    ```
* Implementing the Circuit trait is how you describe both the structure and the 
logic of your custom circuit in Halo2. The trait ensures a clear separation 
between configuration (structure) and synthesis (assignment and constraint 
enforcement), which is essential for secure and flexible zero-knowledge proof 
systems.

#### TODO(sure2web3): There are a lot of other traits that are used above which should be documented before the above crates.

- lookup
- permutation
- Assigned
- Error
- Layouter
- Region
- Rotation
- compress_selectors

#### 13. Value
```mermaid
classDiagram
    class Value~V~ {
        - inner: Option~V~
        + default() Self
        + unknown() Self
        + known(value: V) Self
        + assign() Result~V, Error~
        + as_ref() Value~&V~
        + as_mut() Value~&mut V~
        + into_option() Option~V~
        + assert_if_known~F: FnOnce(&V) -> bool~(f: F)
        + error_if_known_and~F: FnOnce(&V) -> bool~(f: F) Result~（）, Error~
        + map~W, F: FnOnce(V) -> W~(f: F) Value~W~
        + and_then~W, F: FnOnce(V) -> Value< W>~(f: F) Value~W~
        + zip~W~(other: Value~W~) Value<（V, W）>
        + unzip() (Value~V~, Value~W~)
        + copied() Value~V~
        + cloned() Value~V~
        + transpose_array() [Value~V~; LEN]
        + transpose_vec(length: usize) Vec~Value< V>~
        + to_field~F: Field~() Value~Assigned< F>~
        + into_field~F: Field~() Value~Assigned< F>~
        + double~F: Field~() Value~Assigned< F>~
        + square~F: Field~() Value~Assigned< F>~
        + cube~F: Field~() Value~Assigned< F>~
        + invert~F: Field~() Value~Assigned< F>~
    }

    class Assigned~F~ {
        - ...
    }

    class Error {
        - ...
    }

    Value <--|> Assigned : can convert to
    Value --> Error : returns
    Value "1" *-- "1" Option : contains

    %% Arithmetic operations
    Value --|> Neg : implements
    Value --|> Add : implements
    Value --|> Sub : implements
    Value --|> Mul : implements
```

- **Value**: The `Value` struct is a core abstraction for representing values that may or may not be known during Halo2 circuit synthesis. It wraps an Option<V>, but intentionally hides the enum interface to prevent accidental unwrapping and to ensure that unknown (unwitnessed) values propagate safely throughout the circuit logic. `Value` supports arithmetic operations (like addition, subtraction, multiplication, and negation) and functional transformations, always propagating the "unknown" state if any operand is unknown. This design is crucial for zero-knowledge circuits, where some values may not be available at synthesis time, and ensures that circuit constraints and assignments remain sound and secure. The struct also provides utility methods for mapping, zipping, and converting values, making it ergonomic and safe to use in complex circuit construction.

    ```rust
    /// A value that might exist within a circuit.
    ///
    /// This behaves like `Option<V>` but differs in two key ways:
    /// - It does not expose the enum cases, or provide an `Option::unwrap` equivalent. This
    ///   helps to ensure that unwitnessed values correctly propagate.
    /// - It provides pass-through implementations of common traits such as `Add` and `Mul`,
    ///   for improved usability.
    #[derive(Clone, Copy, Debug)]
    pub struct Value<V> {
        inner: Option<V>,
    }
    ```

- **Functions**:
    | Function Signature | Class | Functionality | Principle in Halo2 |
    | --- | --- | --- | --- |
    | `fn default() -> Self` | `Value<V>` | Returns an unwitnessed value by default. | In Halo2, it's often necessary to initialize values in a circuit. This method provides a default initialization with an unwitnessed value. |
    | `fn unknown() -> Self` | `Value<V>` | Constructs an unwitnessed value. | Represents a value that is not yet known in the circuit, which is important for handling cases where the witness is not available. |
    | `fn known(value: V) -> Self` | `Value<V>` | Constructs a known value. | Allows the circuit to represent values that are already known, which can be used in subsequent operations. |
    | `fn assign(self) -> Result<V, Error>` | `Value<V>` | Obtains the inner value for assigning into the circuit. Returns `Error::Synthesis` if the value is unknown. | Ensures that only known values can be assigned to cells in the circuit, preventing synthesis errors. |
    | `fn as_ref(&self) -> Value<&V>` | `Value<V>` | Converts from `&Value<V>` to `Value<&V>`. | Facilitates borrowing the inner value without taking ownership. |
    | `fn as_mut(&mut self) -> Value<&mut V>` | `Value<V>` | Converts from `&mut Value<V>` to `Value<&mut V>`. | Allows mutable borrowing of the inner value. |
    | `fn into_option(self) -> Option<V>` | `Value<V>` | Converts the `Value<V>` to an `Option<V>`. | For internal crate usage, provides a way to access the inner `Option` type. |
    | `fn assert_if_known<F: FnOnce(&V) -> bool>(f: F) -> ()` | `Value<V>` | Enforces an assertion on the contained value if it is known. Ignores the assertion if the value is unknown. | Helps to verify the integrity of known values in the circuit without affecting unknown values. |
    | `fn error_if_known_and<F: FnOnce(&V) -> bool>(f: F) -> Result<(), Error>` | `Value<V>` | Checks the contained value for an error condition if it is known. Ignores the check if the value is unknown. | Allows for error checking on known values in the circuit. |
    | `fn map<W, F: FnOnce(V) -> W>(f: F) -> Value<W>` | `Value<V>` | Maps a `Value<V>` to `Value<W>` by applying a function to the contained value. | Enables transformation of the inner value while maintaining the `Value` wrapper. |
    | `fn and_then<W, F: FnOnce(V) -> Value<W>>(f: F) -> Value<W>` | `Value<V>` | Returns `Value::unknown()` if the value is unknown, otherwise calls `f` with the wrapped value and returns the result. | Allows for conditional processing of the inner value. |
    | `fn zip<W>(other: Value<W>) -> Value<(V, W)>` | `Value<V>` | Zips `self` with another `Value`. Returns `Value::unknown()` if either value is unknown. | Combines two `Value` instances into a single `Value` containing a tuple. |
    | `fn unzip(self) -> (Value<V>, Value<W>)` | `Value<(V, W)>` | Unzips a value containing a tuple of two values. Returns `(Value::unknown(), Value::unknown())` if the value is unknown. | Separates a `Value` containing a tuple into two individual `Value` instances. |
    | `fn copied(self) -> Value<V>` | `Value<&V>` | Maps a `Value<&V>` to a `Value<V>` by copying the contents of the value. | Allows for creating a new `Value` by copying the inner value. |
    | `fn cloned(self) -> Value<V>` | `Value<&V>` | Maps a `Value<&V>` to a `Value<V>` by cloning the contents of the value. | Enables creating a new `Value` by cloning the inner value. |
    | `fn transpose_array(self) -> [Value<V>; LEN]` | `Value<[V; LEN]>` | Transposes a `Value<[V; LEN]>` into a `[Value<V>; LEN]`. Maps `Value::unknown()` to `[Value::unknown(); LEN]`. | Converts an array inside a `Value` to an array of `Value` instances. |
    | `fn transpose_vec(self, length: usize) -> Vec<Value<V>>` | `Value<I>` | Transposes a `Value<impl IntoIterator<Item = V>>` into a `Vec<Value<V>>`. Maps `Value::unknown()` to `vec![Value::unknown(); length]`. | Converts an iterable inside a `Value` to a vector of `Value` instances. |
    | `fn to_field<F: Field>(&self) -> Value<Assigned<F>>` | `Value<V>` | Returns the field element corresponding to this value. | Converts the inner value to a field element represented by `Assigned<F>`. |
    | `fn into_field<F: Field>(self) -> Value<Assigned<F>>` | `Value<V>` | Returns the field element corresponding to this value. | Similar to `to_field`, but consumes the `Value` instance. |
    | `fn double<F: Field>(&self) -> Value<Assigned<F>>` | `Value<V>` | Doubles this field element. | Performs the doubling operation on the inner field element and returns a new `Value` with the result. |
    | `fn square<F: Field>(&self) -> Value<Assigned<F>>` | `Value<V>` | Squares this field element. | Squares the inner field element and returns a new `Value` with the result. |
    | `fn cube<F: Field>(&self) -> Value<Assigned<F>>` | `Value<V>` | Cubes this field element. | Cubes the inner field element and returns a new `Value` with the result. |
    | `fn invert<F: Field>(&self) -> Value<Assigned<F>>` | `Value<V>` | Inverts this assigned value (taking the inverse of zero to be zero). | Inverts the inner field element and returns a new `Value` with the result. |
    | `fn evaluate(self) -> Value<F>` | `Value<Assigned<F>>` | Evaluates this value directly, performing an unbatched inversion if necessary. Returns zero if the denominator is zero. | Computes the actual value of the `Assigned<F>` and returns it as a `Value<F>`. |
    | `fn neg(self) -> Self::Output` | `Value<V>` | Implements the negation operation for `Value<V>`. | Applies the negation operation to the inner value if it is known. |
    | `fn add(self, rhs: Self) -> Self::Output` | `Value<V>` | Implements the addition operation for `Value<V>`. | Adds two `Value` instances if both inner values are known. |
    | `fn sub(self, rhs: Self) -> Self::Output` | `Value<V>` | Implements the subtraction operation for `Value<V>`. | Subtracts two `Value` instances if both inner values are known. |
    | `fn mul(self, rhs: Self) -> Self::Output` | `Value<V>` | Implements the multiplication operation for `Value<V>`. | Multiplies two `Value` instances if both inner values are known. |
    | `fn add(self, rhs: Value<F>) -> Self::Output` | `Value<Assigned<F>>` | Implements the addition operation between `Value<Assigned<F>>` and `Value<F>`. | Adds a `Value<Assigned<F>>` and a `Value<F>` if both inner values are known. |
    | `fn add(self, rhs: F) -> Self::Output` | `Value<Assigned<F>>` | Implements the addition operation between `Value<Assigned<F>>` and a field element `F`. | Adds a `Value<Assigned<F>>` and a field element if the inner value of `Value<Assigned<F>>` is known. |
    | `fn sub(self, rhs: Value<F>) -> Self::Output` | `Value<Assigned<F>>` | Implements the subtraction operation between `Value<Assigned<F>>` and `Value<F>`. | Subtracts a `Value<F>` from a `Value<Assigned<F>>` if both inner values are known. |
    | `fn sub(self, rhs: F) -> Self::Output` | `Value<Assigned<F>>` | Implements the subtraction operation between `Value<Assigned<F>>` and a field element `F`. | Subtracts a field element from a `Value<Assigned<F>>` if the inner value of `Value<Assigned<F>>` is known. |
    | `fn mul(self, rhs: Value<F>) -> Self::Output` | `Value<Assigned<F>>` | Implements the multiplication operation between `Value<Assigned<F>>` and `Value<F>`. | Multiplies a `Value<Assigned<F>>` and a `Value<F>` if both inner values are known. |
    | `fn mul(self, rhs: F) -> Self::Output` | `Value<Assigned<F>>` | Implements the multiplication operation between `Value<Assigned<F>>` and a field element `F`. | Multiplies a `Value<Assigned<F>>` and a field element if the inner value of `Value<Assigned<F>>` is known. |

#### 14. RegionIndex and RegionStart
The `RegionIndex` and `RegionStart` play important roles in this process:
- **`RegionIndex`**: It helps in identifying and referring to different regions. This is useful when multiple regions are defined in a circuit, and the layouter needs to manage them efficiently. For example, during the measurement and assignment phases of the circuit synthesis, the layouter may need to access and manipulate specific regions based on their indices.
- **`RegionStart`**: It determines the starting position of a region within the circuit layout. This is essential for ensuring that different regions do not overlap and that the overall circuit is well - organized. The starting row information is used when assigning cells and constraints within a region.
```mermaid
classDiagram
    class RegionIndex {
        - usize 0
        + from(usize) RegionIndex
        + deref(): &usize
    }

    class RegionStart {
        - usize 0
        + from(usize) RegionStart
        + deref(): &usize
    }

    RegionIndex --|>  From~usize~ : impl
    RegionIndex --|> Deref : impl
    RegionStart --|> From~usize~ : impl
    RegionStart --|> Deref : impl
```
- **RegionIndex**: It serves as an index to uniquely identify a `region` within a layouter. A `layouter` in Halo2 is responsible for organizing and assigning `cells` in a `circuit`, and different regions can be thought of as separate sub - areas within this circuit layout.
    * It is a newtype wrapper around `usize`. The `From<usize>` trait is implemented, allowing easy conversion from a plain `usize` value to a `RegionIndex`. The `Deref` trait is also implemented, which means that a `RegionIndex` can be treated as a reference to a `usize` in many contexts.

    ```rust
    /// Index of a region in a layouter
    #[derive(Clone, Copy, Debug)]
    pub struct RegionIndex(usize);

    impl From<usize> for RegionIndex {
        fn from(idx: usize) -> RegionIndex {
            RegionIndex(idx)
        }
    }

    impl std::ops::Deref for RegionIndex {
        type Target = usize;

        fn deref(&self) -> &Self::Target {
            &self.0
        }
    }
    ```

- **RegionStart**: It represents the starting row of a region in a layouter. This is crucial for determining the position of a region within the overall circuit layout.
    ```rust
    /// Starting row of a region in a layouter
    #[derive(Clone, Copy, Debug, PartialEq, Eq)]
    pub struct RegionStart(usize);

    impl From<usize> for RegionStart {
        fn from(idx: usize) -> RegionStart {
            RegionStart(idx)
        }
    }

    impl std::ops::Deref for RegionStart {
        type Target = usize;

        fn deref(&self) -> &Self::Target {
            &self.0
        }
    }
    ```

#### 15. RegionLayouter
```mermaid
classDiagram
    %% Traits
    class RegionLayouter{
        <<trait>>
        - F: Field
        + enable_selector(annotation, selector, offset) Result<(), Error>
        + assign_advice(annotation, column, offset, to) Result<Cell, Error>
        + assign_advice_from_constant(annotation, column, offset, constant) Result<Cell, Error>
        + assign_advice_from_instance(annotation, instance, row, advice, offset) Result<(Cell, Value<F>), Error>
        + instance_value(instance, row) Result<Value<F>, Error>
        + assign_fixed(annotation, column, offset, to) Result<Cell, Error>
        + constrain_constant(cell, constant) Result<(), Error>
        + constrain_equal(left, right) Result<(), Error>
    }
    
    %% Structs
    class RegionShape{
        <<struct>>
        - region_index: RegionIndex
        - columns: HashSet<RegionColumn>
        - row_count: usize
        + new(region_index) Self
        + region_index() RegionIndex
        + columns() &HashSet<RegionColumn>
        + row_count() usize
    }
    
    class RegionColumn{
        <<enum>>
        - Column(Column<Any>)
        - Selector(Selector)
    }
    
    class Column{
        <<struct>>
        - index: usize
        - column_type: ColumnType
    }
    
    class ColumnType{
        <<enum>>
        - Advice
        - Instance
        - Fixed
    }
    
    class Cell{
        <<struct>>
        - region_index: RegionIndex
        - row_offset: usize
        - column: Column<Any>
    }
    
    class Value{
        <<struct>>
        - inner: Option<V>
        + unknown() Self
        + known(value: V) Self
    }
    
    class Assigned{
        <<struct>>
        - F: Field
        - numerator: F
        - denominator: F
    }
    
    class Selector{
        <<struct>>
        - index: usize
    }
    
    class RegionIndex{
        <<struct>>
        - index: usize
    }
    
    class Error{
        <<enum>>
        - Synthesis
        - InvalidInstances
        - ConstraintSystemFailure
        - ...
    }
    
    class HashSet{
        <<external>>
    }
    
    %% Relationships
    RegionLayouter <|.. RegionShape : implements
    RegionShape --> RegionColumn : columns
    RegionShape --> RegionIndex : region_index
    RegionShape --> HashSet : columns
    RegionShape --> Cell : returns
    
    RegionColumn --> Column : Column variant
    RegionColumn --> Selector : Selector variant
    
    Cell --> RegionIndex : region_index
    Cell --> Column : column
    
    Column --> ColumnType : column_type
    
    Value --> Error : returns
    Value --> Assigned : contains
```

- **RegionLayouter<F>**: A trait that defines the interface for implementing a custom region layouter in a Halo2 circuit. It provides methods for enabling selectors, assigning advice, fixed, and instance columns, and creating equality constraints between cells. This trait abstracts the process of assigning values to columns and enforcing constraints within a region of the circuit.

- **RegionShape**: A struct that represents the shape of a region in a Halo2 circuit. It tracks the region index, the set of columns used by the region (including both concrete columns and selectors), and the number of rows used. This struct is used to analyze the structure of a region without actually assigning values, which is useful for circuit planning and optimization.
    ```rust
    /// The shape of a region. For a region at a certain index, we track
    /// the set of columns it uses as well as the number of rows it uses.
    #[derive(Clone, Debug)]
    pub struct RegionShape {
        pub(super) region_index: RegionIndex,
        pub(super) columns: HashSet<RegionColumn>,
        pub(super) row_count: usize,
    }
    ```
- **RegionColumn**: An enum that represents the types of columns involved in a region. It can be either a concrete column (advice, instance, or fixed) or a selector. This enum allows regions to track both types of columns uniformly.
    ```rust
    /// The virtual column involved in a region. This includes concrete columns,
    /// as well as selectors that are not concrete columns at this stage.
    #[derive(Eq, PartialEq, Copy, Clone, Debug, Hash)]
    pub enum RegionColumn {
        /// Concrete column
        Column(Column<Any>),
        /// Virtual column representing a (boolean) selector
        Selector(Selector),
    }
    ```
- **Functions**:
    | Function Signature | Class/Struct | Functionality | Principle in Halo2 |
    | --- | --- | --- | --- |
    | `fn enable_selector(&mut self, annotation, selector, offset) -> Result<(), Error>` | `RegionLayouter` | Enables a selector at the given offset within the region. | Selectors are used to conditionally enable constraints in specific rows, allowing the circuit to perform different operations based on the input. |
    | `fn assign_advice(&mut self, annotation, column, offset, to) -> Result<Cell, Error>` | `RegionLayouter` | Assigns a value to an advice column at the given offset. | Advice columns store witness values, which are the private inputs to the circuit that need to be proven without revealing. |
    | `fn assign_advice_from_constant(&mut self, annotation, column, offset, constant) -> Result<Cell, Error>` | `RegionLayouter` | Assigns a constant value to an advice column and equality-constrains it. | This allows the circuit to use public constants while maintaining the structure of advice columns. |
    | `fn assign_advice_from_instance(&mut self, annotation, instance, row, advice, offset) -> Result<(Cell, Value<F>), Error>` | `RegionLayouter` | Assigns the value from an instance column to an advice column and equality-constrains them. | Instance columns store public inputs, and this method allows the circuit to reference and use those inputs. |
    | `fn instance_value(&mut self, instance, row) -> Result<Value<F>, Error>` | `RegionLayouter` | Retrieves the value from an instance column at the specified row. | This allows the circuit to access public inputs during synthesis. |
    | `fn assign_fixed(&mut self, annotation, column, offset, to) -> Result<Cell, Error>` | `RegionLayouter` | Assigns a value to a fixed column at the given offset. | Fixed columns store values that are the same across all instances of the circuit, such as constants and precomputed values. |
    | `fn constrain_constant(&mut self, cell, constant) -> Result<(), Error>` | `RegionLayouter` | Constrains a cell to have a specific constant value. | This ensures that a particular cell in the circuit always evaluates to the specified constant. |
    | `fn constrain_equal(&mut self, left, right) -> Result<(), Error>` | `RegionLayouter` | Constrains two cells to have the same value. | This enforces that the values in the two specified cells must be equal, which is a fundamental operation for building circuit constraints. |
    | `fn new(region_index) -> Self` | `RegionShape` | Creates a new `RegionShape` for a region with the given index. | Initializes the region shape with the specified index and empty column and row tracking. |
    | `fn region_index(&self) -> RegionIndex` | `RegionShape` | Returns the region index of the shape. | Provides access to the region index for external use. |
    | `fn columns(&self) -> &HashSet<RegionColumn>` | `RegionShape` | Returns a reference to the set of columns used by the region. | Allows external code to inspect the columns used by the region. |
    | `fn row_count(&self) -> usize` | `RegionShape` | Returns the number of rows used by the region. | Provides information about the size of the region. |

#### 16. Region

```mermaid
classDiagram
    class Region {
        - &'r mut dyn layouter::RegionLayouter~F~ region
        + enable_selector(annotation, selector, offset) Result~（）, Error~
        + assign_advice(annotation, column, offset, to) Result< AssignedCell< VR, F>, Error>
        + assign_advice_from_constant(annotation, column, offset, constant) Result< AssignedCell< VR, F>, Error>
        + assign_advice_from_instance(annotation, instance, row, advice, offset) Result< AssignedCell< F, F>, Error>
        + instance_value(instance, row) Result~Value< F>, Error~
        + assign_fixed(annotation, column, offset, to) Result< AssignedCell< VR, F>, Error>
        + constrain_constant(cell, constant) Result~（）, Error~
        + constrain_equal(left, right) Result~（）, Error~
    }

    class AssignedCell {
        - Value~VR~ value
        - Cell cell
        - PhantomData~F~ _marker
    }

    class Cell {
        - RegionIndex region_index
        - usize row_offset
        - Column~Any~ column
    }

    class Value {
        - ...
    }

    class Column {
        - ...
    }

    class Selector {
        - ...
    }

    Region --> AssignedCell : returns
    Region --> Cell : uses
    Region --> Value : uses
    Region --> Column : uses
    Region --> Selector : uses
    AssignedCell --> Cell : contains
    AssignedCell --> Value : contains
```

- **Region**: A `Region` represents a contiguous block of rows in the circuit where a chip can assign values to cells. Regions allow chips to use relative row offsets for assignments, making circuit construction modular and flexible. The `Layouter` manages these regions and can optimize how they are placed in the final circuit. Regions are essential for organizing assignments, enabling selectors, and enforcing constraints (like equality or constants) between cells.
    * All of the methods implemented on the `Region` struct ultimately delegate their functionality to the underlying `region: &'r mut dyn layouter::RegionLayouter<F>` field. 
    * This means that when you call a method like `assign_advice`, `assign_fixed`, `constrain_equal`, or any other assignment or constraint function on a `Region`, it internally calls the corresponding method on the `RegionLayouter` trait object. 
    * This design allows Region to provide a high-level, chip-friendly API while relying on the lower-level layouter implementation for the actual circuit assignment and constraint logic.

    ```rust 
    /// A region of the circuit in which a [`Chip`] can assign cells.
    ///
    /// Inside a region, the chip may freely use relative offsets; the [`Layouter`] will
    /// treat these assignments as a single "region" within the circuit.
    ///
    /// The [`Layouter`] is allowed to optimise between regions as it sees fit. Chips must use
    /// [`Region::constrain_equal`] to copy in variables assigned in other regions.
    ///
    /// TODO: It would be great if we could constrain the columns in these types to be
    /// "logical" columns that are guaranteed to correspond to the chip (and have come from
    /// `Chip::Config`).
    #[derive(Debug)]
    pub struct Region<'r, F: Field> {
        region: &'r mut dyn layouter::RegionLayouter<F>,
    }
    ```

- **Functions**: 

    | Function Signature | Functionality | Principle in Halo2 |
    | --- | --- | --- |
    | `pub(crate) fn enable_selector<A, AR>(&mut self, annotation: A, selector: &Selector, offset: usize) -> Result<(), Error>` | Enables a selector at a specific row offset in the region, activating certain constraints for that row. | In Halo2, selectors are used to enable or disable certain gates or constraints within a circuit. By enabling a selector at a specific offset, the circuit can control which parts of the computation are active at different rows. This is crucial for constructing efficient and flexible circuits. |
    | `pub fn assign_advice<'v, V, VR, A, AR>(&'v mut self, annotation: A, column: Column<Advice>, offset: usize, mut to: V) -> Result<AssignedCell<VR, F>, Error>` | Assigns a value to an advice column at a specific offset, returning an AssignedCell with the value and cell location. | Advice columns in Halo2 are used to store witness values, which are private inputs to the circuit. This function allows the circuit to assign a value to an advice column at a specific location. The `to` function is used to compute the value, and it is guaranteed to be called at most once. The result is an `AssignedCell` containing the assigned value and the cell's location. |
    | `pub fn assign_advice_from_constant<VR, A, AR>(&mut self, annotation: A, column: Column<Advice>, offset: usize, constant: VR) -> Result<AssignedCell<VR, F>, Error>` | Assigns a constant value to the advice column at the given offset within the region. | Constant values in Halo2 are assigned to fixed columns and then equality-constrained to advice columns. This function simplifies the process of assigning a constant value to an advice column. The constant value is first assigned to a fixed column configured via `ConstraintSystem::enable_constant`, and then the advice column is constrained to have the same value. |
    | `pub fn assign_advice_from_instance<A, AR>(&mut self, annotation: A, instance: Column<Instance>, row: usize, advice: Column<Advice>, offset: usize) -> Result<AssignedCell<F, F>, Error>` | Assigns the value of an instance column's cell at an absolute location to an advice column at the given offset within the region. | Instance columns in Halo2 are used to store public inputs to the circuit. This function allows the circuit to copy a public input value from an instance column to an advice column. The result is an `AssignedCell` containing the copied value and the cell's location. |
    | `pub fn instance_value(&mut self, instance: Column<Instance>, row: usize) -> Result<Value<F>, Error>` | Returns the value of the instance column's cell at the absolute location `row`. | This function provides a convenient way to access the value of an instance column's cell. However, it does not create any constraints. To use the instance value in the circuit, callers still need to use `assign_advice_from_instance` to copy the value to an advice column and create the necessary constraints. |
    | `pub fn assign_fixed<'v, V, VR, A, AR>(&'v mut self, annotation: A, column: Column<Fixed>, offset: usize, mut to: V) -> Result<AssignedCell<VR, F>, Error>` | Assigns a value to a fixed column at a specific offset within the region. | Fixed columns in Halo2 are used to store values that are the same for all instances of the circuit. This function allows the circuit to assign a value to a fixed column at a specific location. The `to` function is used to compute the value, and it is guaranteed to be called at most once. The result is an `AssignedCell` containing the assigned value and the cell's location. |
    | `pub fn constrain_constant<VR>(&mut self, cell: Cell, constant: VR) -> Result<(), Error>` | Constrains a cell to have a constant value. | In Halo2, constraints are used to enforce relationships between cells. This function adds a constraint to ensure that a given cell has a specific constant value. If the cell is in a column where equality has not been enabled, an error will be returned. |
    | `pub fn constrain_equal(&mut self, left: Cell, right: Cell) -> Result<(), Error>` | Constrains two cells to have the same value. | This function adds a constraint to ensure that two given cells have the same value. If either of the cells is in a column where equality has not been enabled, an error will be returned. Constraining cells to be equal is a fundamental operation in constructing circuits in Halo2. |

#### 17. Cell and AssignedCell
```mermaid
classDiagram
    class Cell {
        - RegionIndex region_index
        - usize row_offset
        - Column~Any~ column
    }

    class AssignedCell {
        - Value~V~ value
        - Cell cell
        - PhantomData~F~ _marker
        + value() Value~&V~
        + cell() Cell
        + value_field() Value~Assigned< F>~
        + evaluate() AssignedCell~F, F~
        + copy_advice(annotation, region, column, offset) Result~Self, Error~
    }

    AssignedCell "1" -- "1" Cell : contains
```

- **Cell**: A `Cell` is a pointer to a cell within a circuit, identified by a region index, row offset, and column. This information is crucial for precisely locating and referring to cells during circuit construction and synthesis.

    ```rust
    /// A pointer to a cell within a circuit.
    #[derive(Clone, Copy, Debug)]
    pub struct Cell {
        /// Identifies the region in which this cell resides.
        region_index: RegionIndex,
        /// The relative offset of this cell within its region.
        row_offset: usize,
        /// The column of this cell.
        column: Column<Any>,
    }
    ```

- **AssignedCell**: An `AssignedCell` represents a cell that has been assigned a value within the circuit. It contains the value (`Value<V>`) assigned to the cell and the `Cell` itself. The `PhantomData<F>` is used to indicate the field type `F` without actually storing an instance of it. It provides methods to access the value and copy the value to another advice cell with equality constraints. 
    ```rust
    /// An assigned cell.
    #[derive(Clone, Debug)]
    pub struct AssignedCell<V, F: Field> {
        value: Value<V>,
        cell: Cell,
        _marker: PhantomData<F>,
    }
    ```

- **Functions**:

    | Function Signature | Functionality | Principle or Role in Halo2 |
    | --- | --- | --- |
    | `pub fn value(&self) -> Value<&V>` | Returns a reference to the value of the `AssignedCell`. | In Halo2, cells are the basic units for storing values in a circuit. This function allows users to access the value assigned to a specific cell. It provides a way to read the value without taking ownership, which is important for maintaining the integrity of the circuit's state. The `Value` type can represent either a known value or an unknown value, which is useful during the circuit construction process where some values may not be known until later. |
    | `pub fn cell(&self) -> Cell` | Returns the `Cell` that the assigned cell refers to. | The `Cell` struct contains information about the location of the cell within the circuit, such as the region index, row offset, and column. This function allows users to retrieve this location information, which is crucial for performing operations like cell constraints and value assignments in the correct context. In Halo2, the layout of cells in regions and columns is carefully managed to ensure the circuit's correctness and efficiency. |
    | `pub fn value_field(&self) -> Value<Assigned<F>> where for<'v> Assigned<F>: From<&'v V>` | Returns the field element value of the `AssignedCell`. | In cryptographic circuits, values are often represented as elements of a finite field. This function converts the value of the `AssignedCell` to a field element (`Assigned<F>`). The `Assigned` type is used to represent values that have been assigned to cells, and it can handle values in different forms (e.g., zero, trivial, or rational). This conversion is necessary for performing arithmetic operations and enforcing constraints within the circuit. |
    | `pub fn evaluate(self) -> AssignedCell<F, F>` | Evaluates this assigned cell's value directly, performing an unbatched inversion if necessary. If the denominator is zero, the returned cell's value is zero. | In some cryptographic protocols, values may be represented as fractions to enable batch inversion for efficiency. However, in certain cases, an unbatched inversion may be required. This function evaluates the cell's value, performing the necessary inversion if the value is represented as a fraction. If the denominator is zero, it ensures that the resulting value is zero, which is a common convention in field arithmetic. This evaluation step is important for obtaining the final value that can be used in subsequent operations in the circuit. |
    | `pub fn copy_advice<A, AR>(&self, annotation: A, region: &mut Region<'_, F>, column: Column<Advice>, offset: usize) -> Result<Self, Error> where A: Fn() -> AR, AR: Into<String>, V: Clone, for<'v> Assigned<F>: From<&'v V>` | Copies the value to a given advice cell and constrains them to be equal. Returns an error if either this cell or the given cell are in columns where equality has not been enabled. | In Halo2, advice columns are used to store witness values that are not publicly known. This function allows users to copy the value of an existing `AssignedCell` to a new advice cell. It also enforces an equality constraint between the two cells, ensuring that they have the same value. This is useful for creating redundant copies of values or for propagating values between different parts of the circuit. The annotation is used to provide a descriptive name for the operation, which can be helpful for debugging and auditing the circuit. |

#### 18. TableLayouter and Table
In Halo2, `lookup tables` are used to efficiently enforce that certain values in the circuit belong to a predefined set (the table). This is crucial for range checks, bit decompositions, and other constraints that require membership in a set. The `TableLayouter` trait abstracts the logic for assigning values to these tables, while the `Table` struct provides a user-friendly API for chips to interact with lookup tables during circuit synthesis.
```mermaid
classDiagram
    class TableLayouter~F~ {
        + assign_cell(annotation, column, offset, to) Result<（）, Error>
    }

    class Table~F~ {
        - &'r mut dyn TableLayouter~F~ table
        + assign_cell(annotation, column, offset, to) Result<（）, Error>
    }

    TableLayouter <|.. Table : implements
    Table --> TableLayouter : uses

    class Value~VR~ {
        - ...
    }

    class Assigned~F~ {
        - ...
    }

    class TableColumn {
        - ...
    }

    class Error {
        - ...
    }

    Table --> Value : uses
    Table --> Assigned : uses
    Table --> TableColumn : uses
    Table --> Error : returns
    TableLayouter --> Value : uses
    TableLayouter --> Assigned : uses
    TableLayouter --> TableColumn : uses
    TableLayouter --> Error : returns
```

- **TableLayouter**: The `TableLayouter<F>` is a helper trait designed for implementing a custom `Layouter`. It is specifically used for implementing table assignments. This trait provides a single method `assign_cell` that allows assigning a fixed value to a table cell. If the table cell has already been assigned, an error will be returned.
    * Defines the interface for assigning values to `lookup table` cells in the circuit.
    ```rust
    /// Helper trait for implementing a custom [`Layouter`].
    ///
    /// This trait is used for implementing table assignments.
    ///
    /// [`Layouter`]: super::Layouter
    pub trait TableLayouter<F: Field>: std::fmt::Debug {
        /// Assigns a fixed value to a table cell.
        ///
        /// Returns an error if the table cell has already been assigned to.
        fn assign_cell<'v>(
            &'v mut self,
            annotation: &'v (dyn Fn() -> String + 'v),
            column: TableColumn,
            offset: usize,
            to: &'v mut (dyn FnMut() -> Value<Assigned<F>> + 'v),
        ) -> Result<(), Error>;
    }
    ```

- **Table**: The `Table<F>` represents a lookup table in the circuit. It holds a mutable reference to a `TableLayouter<F>`. The `Table<F>` provides a method `assign_cell` that delegates the assignment operation to the underlying `TableLayouter<F>`. It also ensures that the value returned by the `to` function is converted to the appropriate field type.
    * Provides a high-level, ergonomic API for chips to assign values to `lookup tables`.
    ```rust
    /// A lookup table in the circuit.
    #[derive(Debug)]
    pub struct Table<'r, F: Field> {
        table: &'r mut dyn TableLayouter<F>,
    }
    ```

- **Functions**:
    | Function Signature | Functionality | Principle in Halo2 |
    | --- | --- | --- |
    | in `TableLayouter<F>`<br><br> `fn assign_cell<'v>(&'v mut self, annotation: &'v (dyn Fn() -> String + 'v), column: TableColumn, offset: usize, to: &'v mut (dyn FnMut() -> Value<Assigned<F>> + 'v)) -> Result<(), Error>` | Assigns a fixed value to a table cell. Returns an error if the table cell has already been assigned to. | In Halo2, tables are used to perform lookup operations. The `assign_cell` method is used to populate the table with fixed values. Each cell in the table can only be assigned once to ensure the integrity of the table data. |
    | in `Table<F>`<br><br> `pub fn assign_cell<'v, V, VR, A, AR>(&'v mut self, annotation: A, column: TableColumn, offset: usize, mut to: V) -> Result<(), Error>` | Assigns a fixed value to a table cell. Returns an error if the table cell has already been assigned to. The `to` function is guaranteed to be called at most once. | This method in `Table<F>` is a wrapper around the `assign_cell` method in `TableLayouter<F>`. It takes a more flexible closure `to` that returns a `Value<VR>`, and converts it to a `Value<Assigned<F>>` before passing it to the underlying `TableLayouter<F>`. This allows for more convenient value assignment in the context of the table. |

#### 19. SimpleTableLayouter

- **SimpleTableLayouter**: A concrete implementation of the `TableLayouter<F>` trait designed for simple table assignments in Halo2 circuits. It manages the assignment of fixed values to table cells and tracks which cells have been assigned.
    ```rust
    /// This type alias is used to track the default value (row 0) for each table column.
    type DefaultTableValue<F> = Option<Value<Assigned<F>>>;

    pub(crate) struct SimpleTableLayouter<'r, 'a, F: Field, CS: Assignment<F> + 'a> {
        /// A mutable reference to the constraint system used for assigning values.
        cs: &'a mut CS,
        /// A reference to an array of table columns that have already been used, preventing reuse.
        used_columns: &'r [TableColumn],
        /// maps from a fixed column to a pair (default value, vector saying which rows are assigned)
        pub(crate) default_and_assigned: HashMap<TableColumn, (DefaultTableValue<F>, Vec<bool>)>,
    }
    ```

- **Functions**:

    | Function Signature | Struct | Functionality | Principle in Halo2 |
    | --- | --- | --- | --- |
    | `pub(crate) fn new(cs: &'a mut CS, used_columns: &'r [TableColumn]) -> Self` | `SimpleTableLayouter` | Creates a new `SimpleTableLayouter` instance. | Initializes the layouter with a reference to the constraint system and an array of used columns to prevent reuse. |
    | `fn assign_cell<'v>(&'v mut self, annotation: &'v (dyn Fn() -> String + 'v), column: TableColumn, offset: usize, to: &'v mut (dyn FnMut() -> Value<Assigned<F>> + 'v)) -> Result<(), Error>` | `SimpleTableLayouter` | Assigns a fixed value to a table cell at the specified offset. | Checks if the column has already been used and returns an error if it has.<br><br> Assigns the value to the constraint system using `assign_fixed`.<br><br> Sets the default value for the column if it's the first assignment at row 0.<br><br> Tracks which rows have been assigned using a boolean vector. |
    | `pub(crate) fn compute_table_lengths<F: Debug>(default_and_assigned: &HashMap<TableColumn, (DefaultTableValue<F>, Vec<bool>)>) -> Result<usize, Error>` | Module Function | Computes the length of a table by ensuring all columns have the same length and are fully assigned. | Verifies that each column has a default value and all rows are assigned.<br><br> Checks that all columns have the same length.<br><br> Returns the length of the table if all checks pass, otherwise returns an error. |

#### 19. Layouter and NamespacedLayouter
```mermaid
classDiagram
    class Layouter~F~ {
        + type Root: Layouter~F~
        + assign_region(name, assignment) Result~AR, Error~
        + assign_table(name, assignment) Result~（）, Error~
        + constrain_instance(cell, column, row) Result~（）, Error~
        + get_root() &mut Self::Root
        + push_namespace(name_fn)
        + pop_namespace(gadget_name)
        + namespace(name_fn) NamespacedLayouter< '_, F, Self::Root>
    }

    class NamespacedLayouter~'a, F, L~ {
        - &'a mut L layouter
        - PhantomData~F~
        + type Root = L::Root
        + assign_region(name, assignment) Result~AR, Error~
        + assign_table(name, assignment) Result~（）, Error~
        + constrain_instance(cell, column, row) Result~（）, Error~
        + get_root() &mut Self::Root
        + push_namespace(name_fn) panic
        + pop_namespace(gadget_name) panic
        + drop()
    }

    Layouter <|-- NamespacedLayouter : implements
    NamespacedLayouter --> Layouter : borrows

    class Region~F~ {
        - ...
    }

    class Table~F~ {
        - ...
    }

    class Cell {
        - region_index: RegionIndex
        - row_offset: usize
        - column: Column<Any>
    }

    class Column~Instance~ {
        - ...
    }

    class Error {
        - ...
    }

    Layouter --> Region : uses
    Layouter --> Table : uses
    Layouter --> Cell : uses
    Layouter --> Column~Instance~ : uses
    Layouter --> Error : returns
    NamespacedLayouter --> Region : uses
    NamespacedLayouter --> Table : uses
    NamespacedLayouter --> Cell : uses
    NamespacedLayouter --> Column~Instance~ : uses
    NamespacedLayouter --> Error : returns
```

- **Layouter**: The `Layouter<F>` is a central abstraction that manages how circuit regions and lookup tables are assigned and organized. It is responsible for handling row indices, managing namespaces (for modular circuit design), and ensuring that assignments and constraints are applied correctly and efficiently. The layouter is chip-agnostic, meaning it can be used with any chip implementation, and it provides the flexibility to optimize circuit layout and assignment strategies. It provides methods for assigning regions and tables, constraining instances, managing namespaces, and getting the root of the assignment.
    * The Layouter trait defines the interface for assigning regions and tables, constraining instance values, and managing namespaces within a Halo2 circuit.
    * It abstracts over the details of row management and circuit assignment, allowing chips to focus on logic rather than layout.

- **NamespacedLayouter**: The `NamespacedLayouter<'a, F, L>` is a wrapper around a `Layouter<F>` that manages namespaces. It borrows a `Layouter` and pushes a namespace context when created. When dropped, it pops out of the namespace context. It implements the `Layouter<F>` trait, delegating most of the operations to the underlying `Layouter`, but it panics if `push_namespace` or `pop_namespace` is called directly, as these operations should be handled by the root `Layouter`.
    * NamespacedLayouter wraps a mutable reference to a layouter and manages entering and exiting a namespace context. When dropped, it automatically pops the namespace, ensuring correct scoping.

- **Functions**: 
    | Function Signature | Class | Functionality | Principle in Halo2 |
    | --- | --- | --- | --- |
    | `fn assign_region<A, AR, N, NR>(&mut self, name: N, assignment: A) -> Result<AR, Error>` | `Layouter<F>` | Assigns a region of gates to an absolute row number.<br><br> Inside the closure, the chip may freely use relative offsets; the `Layouter` will treat these assignments as a single "region" within the circuit. Outside this closure, the `Layouter` is allowed to optimise as it sees fit. | In Halo2, regions are used to organize the circuit layout. The `Layouter` determines the absolute position of the region and assigns cells within it. This method allows the circuit designer to define the operations within a region. |
    | `fn assign_table<A, N, NR>(&mut self, name: N, assignment: A) -> Result<(), Error>` | `Layouter<F>` | Assigns a table region to an absolute row number. | Tables in Halo2 are used for lookup operations. This method allows the circuit designer to assign values to table cells and determine the position of the table in the circuit. |
    | `fn constrain_instance(&mut self, cell: Cell, column: Column<Instance>, row: usize) -> Result<(), Error>` | `Layouter<F>` | Constrains a `Cell` to equal an instance column's row value at an absolute position. | Instance columns in Halo2 hold public inputs. This method ensures that a cell in the circuit has the same value as a specific row in an instance column, enforcing the relationship between public inputs and the circuit. |
    | `fn get_root(&mut self) -> &mut Self::Root` | `Layouter<F>` | Gets the "root" of this assignment, bypassing the namespacing.<br><br> In this context, "bypassing the namespacing" means accessing the original, top-level layouter directly, without considering any nested namespace layers that may have been created for modularity or organization.<br><br> When you call get_root, you get a reference to the root layouter, ignoring any intermediate or current namespace contexts. This is useful for internal management but is not intended for regular use in circuit code, since you usually want to respect the current namespace structure for clarity and correctness. | This method is used to access the root `Layouter` directly, which is useful for operations that need to be performed at the top level of the assignment hierarchy. |
    | `fn push_namespace<NR, N>(&mut self, name_fn: N)` | `Layouter<F>` | Creates a new (sub)namespace and enters into it. | Namespaces in Halo2 are used to organize the circuit layout and group related operations. This method allows the circuit designer to create a new namespace and start a new scope of operations. |
    | `fn pop_namespace(&mut self, gadget_name: Option<String>)` | `Layouter<F>` | Exits out of the existing namespace. | This method is used to end the current namespace scope and return to the previous one. It can optionally record the name of a gadget associated with the namespace. |
    | `fn namespace<NR, N>(&mut self, name_fn: N) -> NamespacedLayouter<'_, F, Self::Root>` | `Layouter<F>` | Enters into a namespace and returns a `NamespacedLayouter`. | This is a convenience method that creates a new `NamespacedLayouter` and enters a new namespace. It simplifies the process of working with namespaces in the circuit. |
    | `fn assign_region<A, AR, N, NR>(&mut self, name: N, assignment: A) -> Result<AR, Error>` | `NamespacedLayouter<'a, F, L>` | Delegates the region assignment to the underlying `Layouter`. | The `NamespacedLayouter` simply passes the region assignment operation to the `Layouter` it borrows, maintaining the same behavior as the root `Layouter`. |
    | `fn assign_table<A, N, NR>(&mut self, name: N, assignment: A) -> Result<(), Error>` | `NamespacedLayouter<'a, F, L>` | Delegates the table assignment to the underlying `Layouter`. | Similar to `assign_region`, this method passes the table assignment operation to the underlying `Layouter`. |
    | `fn constrain_instance(&mut self, cell: Cell, column: Column<Instance>, row: usize) -> Result<(), Error>` | `NamespacedLayouter<'a, F, L>` | Delegates the instance constraint to the underlying `Layouter`. | This method ensures that the instance constraint operation is handled by the root `Layouter`. |
    | `fn get_root(&mut self) -> &mut Self::Root` | `NamespacedLayouter<'a, F, L>` | Delegates the root retrieval to the underlying `Layouter`. | It allows access to the root `Layouter` through the `NamespacedLayouter`. |
    | `fn push_namespace<NR, N>(&mut self, _name_fn: N)` | `NamespacedLayouter<'a, F, L>` | Panics if called, as only the root's `push_namespace` should be called. | This is to ensure that namespace creation is properly managed at the root level. |
    | `fn pop_namespace(&mut self, _gadget_name: Option<String>)` | `NamespacedLayouter<'a, F, L>` | Panics if called, as only the root's `pop_namespace` should be called. | This enforces the correct namespace management by preventing direct calls to `pop_namespace` on the `NamespacedLayouter`. |
    | `fn drop(&mut self)` | `NamespacedLayouter<'a, F, L>` | Pops out of the namespace context when the `NamespacedLayouter` is dropped. | This ensures that the namespace is properly exited when the `NamespacedLayouter` goes out of scope. |

#### 20. Chip
- **Chip Trait**: The `Chip` trait represents a set of instructions used by gadgets in the circuit. It stores configuration and loaded state information required during circuit synthesis. The `Config` type holds the configuration for the chip, which can be derived during the `Circuit::configure` phase. The `Loaded` type holds any general chip state that needs to be loaded at the start of `Circuit::synthesize`.

    ```rust
    /// The chip stores state that is required at circuit synthesis time in
    /// [`Chip::Config`], which can be fetched via [`Chip::config`].
    ///
    /// The chip also loads any fixed configuration needed at synthesis time
    /// using its own implementation of `load`, and stores it in [`Chip::Loaded`].
    /// This can be accessed via [`Chip::loaded`].
    pub trait Chip<F: Field>: Sized {
        /// A type that holds the configuration for this chip, and any other state it may need
        /// during circuit synthesis, that can be derived during [`Circuit::configure`].
        ///
        /// [`Circuit::configure`]: crate::plonk::Circuit::configure
        type Config: fmt::Debug + Clone;

        /// A type that holds any general chip state that needs to be loaded at the start of
        /// [`Circuit::synthesize`]. This might simply be `()` for some chips.
        ///
        /// [`Circuit::synthesize`]: crate::plonk::Circuit::synthesize
        type Loaded: fmt::Debug + Clone;

        /// The chip holds its own configuration.
        fn config(&self) -> &Self::Config;

        /// Provides access to general chip state loaded at the beginning of circuit
        /// synthesis.
        ///
        /// Panics if called before `Chip::load`.
        fn loaded(&self) -> &Self::Loaded;
    }
    ```
    * Chips are the building blocks of Halo2 circuits. Each chip defines its own configuration and logic, and can be used by gadgets (higher-level abstractions) to build complex circuits.
    * The `Config` type allows each chip to specify what columns and selectors it needs.
    * The `Loaded` type allows chips to store any precomputed or loaded data needed during synthesis.
    * The trait methods provide access to this configuration and state, ensuring encapsulation and modularity.

### Module Dependencies
- The `circuit.rs` file depends on the `plonk/circuit.rs` for column types, selectors, and other basic circuit components.
- The `circuit/layouter.rs` and `circuit/value.rs` are utility modules that support the main circuit functionality defined in `circuit.rs`. For example, `circuit/layouter.rs` may provide the implementation of the `Layouter` trait, which is used in `Region` to perform actual assignments, and `circuit/value.rs` provides operations on `Value` types.

In summary, the project starts from defining basic types such as column types and selectors, then builds up more complex structures like regions and chips, and finally provides the ability to assign values and enforce constraints in the circuit.