
**Date:** 06 Apr 2026  

**Context:** Writing a unit test for an Airflow Extractor class. Wanted to verify that task_instance.xcom_push is called on failure, but the test failed with an assertion error even though the method is executed in the code.

**Version:** Python 3.11, pytest 9.0.1, Airflow 3.0.6

## Error
```[python]
AssertionError: Expected 'xcom_push' to have been called.
```

## Code (Before)
```[python]
@pytest.fixture
def mock_context():
    """Create a mock Airflow context."""
    context = MagicMock()
    context.get.side_effect = lambda key: {
        "ds": "2026-01-15",
        "task_instance": MagicMock(task_id="extract_studies", try_number=1),
        "dag": MagicMock(dag_id="clinical_trials_etl"),
        "run_id": "manual__2026-01-15T00:00:00+00:00",
    }.get(key)
    context.__getitem__ = lambda self, key: {
        "ds": "2026-01-15",
        "task_instance": MagicMock(task_id="extract_studies", try_number=1),
        "dag": MagicMock(dag_id="clinical_trials_etl"),
        "run_id": "manual__2026-01-15T00:00:00+00:00",
    }.get(key)
    return context
```

## Solution
```[python]
@pytest.fixture
def mock_context():
    """Create a stable Airflow context mock."""
    mock_ti = MagicMock(task_id="extract_studies", try_number=1)
    mock_dag = MagicMock(dag_id="clinical_trials_etl")

    context_dict = {
        "ds": "2026-01-15",
        "task_instance": mock_ti,
        "dag": mock_dag,
        "run_id": "manual__2026-01-15T00:00:00+00:00",
    }

    context = MagicMock()
    context.get.side_effect = lambda key: context_dict.get(key)
    context.__getitem__.side_effect = lambda key: context_dict.get(key)

    return context
```

## Explanation
- **What caused the error** - Each access to self.context["task_instance"] in the Extractor created a new MagicMock instance due to the lambda returning a fresh MagicMock each time. THe test was asserting against a different instance than the one actually used by the code, so xcom_push.assert_called() failed.
- **Why the solution works** - single stable MagicMock for task_instance and always return the same object in get or __getitem__. In the solution, the test and the code share the same instance, so calls are tracked correctly.
- **Key takeaways** - Mocks that maintain state (like Airflow task_instance) must be stable and not recreated dynamically.


---